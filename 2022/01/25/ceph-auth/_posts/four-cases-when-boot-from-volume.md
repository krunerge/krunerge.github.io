---
title: 卷启动虚拟机的四种情况
date: 2018-10-05 16:18:10
tags: openstack nova cinder glance
---

# 卷启动虚拟机
卷启动虚拟机的使用方式有很多，这里通过看代码列出四种使用场景
## 1.glance直接接ceph，使用images池
- 镜像在ceph的images池中，其实我们可以直接通过ceph的clone拷贝镜像的rbd到volumes（cinder池）池中就可以，不过这要把glance的show_multiple_locations和show_dirct_url两个配置打开，这两个配置主要是用来输出image的时候把在ceph上面的位置信息也输出来
- 代码如下：
```
cinder/volume/flows/manager/create_volume.py
_create_from_image
def _create_from_image(self, context, volume,
                       image_location, image_id, image_meta,
                       image_service, **kwargs):
    if (CONF.image_conversion_dir and not
            os.path.exists(CONF.image_conversion_dir)):
        os.makedirs(CONF.image_conversion_dir)
    try:
        image_utils.check_available_space(CONF.image_conversion_dir,
                                          image_meta['size'], image_id)
    except exception.ImageTooBig as err:
        with excutils.save_and_reraise_exception():
            self.message.create(
                context,
                message_field.Action.COPY_IMAGE_TO_VOLUME,
                resource_uuid=volume.id,
                detail=message_field.Detail.NOT_ENOUGH_SPACE_FOR_IMAGE,
                exception=err)

    virtual_size = image_meta.get('virtual_size')
    if virtual_size:
        virtual_size = image_utils.check_virtual_size(virtual_size,
                                                      volume.size,
                                                      image_id)

    volume_is_encrypted = volume.encryption_key_id is not None
    cloned = False
    model_update = None
    if not volume_is_encrypted:
        model_update, cloned = self.driver.clone_image(context,
                                                       volume,
                                                       image_location,
                                                       image_meta,
                                                       image_service)

    # Try and clone the image if we have it set as a glance location.
    if not cloned and 'cinder' in CONF.allowed_direct_url_schemes:
        model_update, cloned = self._clone_image_volume(context,
                                                        volume,
                                                        image_location,
                                                        image_meta)

    # Try and use the image cache, and download if not cached.
    if not cloned:
        model_update = self._create_from_image_cache_or_download(
            context,
            volume,
            image_location,
            image_id,
            image_meta,
            image_service)

    self._handle_bootable_volume_glance_meta(context, volume,
                                             image_id=image_id,
                                             image_meta=image_meta)
    return model_update
```
其中clone_image就是上面第一种情况
```
def clone_image(self, context, volume,
                image_location, image_meta,
                image_service):
    if image_location:
        # Note: image_location[0] is glance image direct_url.
        # image_location[1] contains the list of all locations (including
        # direct_url) or None if show_multiple_locations is False in
        # glance configuration.
        if image_location[1]:
            url_locations = [location['url'] for
                             location in image_location[1]]
        else:
            url_locations = [image_location[0]]

        # iterate all locations to look for a cloneable one.
        for url_location in url_locations:
            if url_location and self._is_cloneable(
                    url_location, image_meta):
                _prefix, pool, image, snapshot = \
                    self._parse_location(url_location)
                volume_update = self._clone(volume, pool, image, snapshot)
                volume_update['provider_location'] = None
                self._resize(volume)
                return volume_update, True
    return ({}, False)
```
直接就是分析镜像的location，然后对镜像的snap镜像clone，很快，秒级

## 2.glance不直接对接ceph，而是对接cinder，这个是glance新支持的
- 这个不仅需要把glance的两个配置打开，还要配置cinder的配置：
```
allowed_direct_url_schemes = cinder
glance_api_version = 2
image_upload_use_cinder_backend = true
image_upload_use_internal_tenant = true
```
- 代码就是如下的分支：
```
if not cloned and 'cinder' in CONF.allowed_direct_url_schemes:
    model_update, cloned = self._clone_image_volume(context,
                                                    volume,
                                                    image_location,
                                                    image_meta)
```
```
def _clone_image_volume(self, context, volume, image_location, image_meta):
    """Create a volume efficiently from an existing image.

    Returns a dict of volume properties eg. provider_location,
    boolean indicating whether cloning occurred
    """
    # NOTE (lixiaoy1): currently can't create volume from source vol with
    # different encryptions, so just return.
    if not image_location or volume.encryption_key_id:
        return None, False

    if (image_meta.get('container_format') != 'bare' or
            image_meta.get('disk_format') != 'raw'):
        LOG.info("Requested image %(id)s is not in raw format.",
                 {'id': image_meta.get('id')})
        return None, False

    image_volume = None
    direct_url, locations = image_location
    urls = set([direct_url] + [loc.get('url') for loc in locations or []])
    image_volume_ids = [url[9:] for url in urls
                        if url and url.startswith('cinder://')]
    image_volumes = self.db.volume_get_all_by_host(
        context, volume['host'], filters={'id': image_volume_ids})

    for image_volume in image_volumes:
        # For the case image volume is stored in the service tenant,
        # image_owner volume metadata should also be checked.
        image_owner = None
        volume_metadata = image_volume.get('volume_metadata') or {}
        for m in volume_metadata:
            if m['key'] == 'image_owner':
                image_owner = m['value']
        if (image_meta['owner'] != volume['project_id'] and
                image_meta['owner'] != image_owner):
            LOG.info("Skipping image volume %(id)s because "
                     "it is not accessible by current Tenant.",
                     {'id': image_volume.id})
            continue

        LOG.info("Will clone a volume from the image volume "
                 "%(id)s.", {'id': image_volume.id})
        break
    else:
        LOG.debug("No accessible image volume for image %(id)s found.",
                  {'id': image_meta['id']})
        return None, False

    try:
        ret = self.driver.create_cloned_volume(volume, image_volume)
        self._cleanup_cg_in_volume(volume)
        return ret, True
    except (NotImplementedError, exception.CinderException):
        LOG.exception('Failed to clone image volume %(id)s.',
                      {'id': image_volume['id']})
        return None, False
```
```
create_cloned_volume
判断cinder的rbd_max_clone_depth配置：
  1. 如果<=0，则使用rbd copy创建新卷
  2. 如果>0, 同时当前缓存卷的深度==rbd_max_clone_depth，则要先做flatten操作，然后在做snap加clone
  3. 如果>0, 同时当前缓存卷的深度<rbd_max_clone_depth, 则直接snap加clone
```
从cinder卷中取出卷id，然后也是clone，很快，秒级

### 3.对镜像做cache
- 对镜像做cache，可以设置把启动的卷缓存在某一个租户下面，这样以后再从这个镜像使用卷启动直接就可以从缓存的卷做快照加clone，还是很快的，秒级，不过如果缓存中没有，也就是第一次使用这个镜像从卷启动，则就是常规流程，先把镜像下载到本地，如果不是raw格式还要做一下转化，然后在上传到创建的空的rbd中，使用的是rbd import命令
- 需要增加的配置
```
cinder_internal_tenant_project_id = 13d54e99646f4d04a52323576d35ab44
cinder_internal_tenant_user_id = 65c9479b8bc24869b7bf441a147d8404
allowed_direct_url_schemes = cinder
image_upload_use_internal_tenant = true
image_volume_cache_enabled = true
image_volume_cache_max_size_gb = 0
image_volume_cache_max_count = 0
```
- 代码分析
```
if not cloned:
    model_update = self._create_from_image_cache_or_download(
        context,
        volume,
        image_location,
        image_id,
        image_meta,
        image_service)
```
  - 第一次使用镜像从卷启动:
    - 下载镜像到本地临时目录（可以配置）
    - copy数据到volume
    - 在从这个volume创建一个缓存volume（在同一个池）
    - 把这个缓存volume加入到cache数据库中
  - cache中有缓存了：
    - 从cache取出缓存volume
    - 判断cinder的rbd_max_clone_depth配置：
      1. 如果<=0，则使用rbd copy创建新卷
      2. 如果>0, 同时当前缓存卷的深度==rbd_max_clone_depth，则要先做flatten操作，然后在做snap加clone
      3. 如果>0, 同时当前缓存卷的深度<rbd_max_clone_depth, 则直接snap加clone

### 4.常规流程，下载到本地
下载到本地的流程跟上面3情况的"第一次使用镜像从卷启动"差不多，只是不要记录到cache数据库
- 下载镜像到本地临时目录（可以配置）
- copy数据到volume
- 在从这个volume创建一个缓存volume

由于笔者不能开启glance中location的配置，所以选择做镜像卷做缓存还是比较好的
