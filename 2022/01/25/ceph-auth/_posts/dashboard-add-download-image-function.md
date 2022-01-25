---
title: dashboard下载镜像
date: 2018-12-20 21:51:05
tags: openstack dashboard horizon
---

# 问题
最近有一个功能，需要在dashboard增加下载镜像的功能，我们都知道，可以通过dashboard上传镜像，可以通过命令下载镜像，但是不能通过dashboard上传镜像

# 解决方法
社区是有一个bp，传输门如下：
https://review.openstack.org/#/c/74799/
但是这个bp是基于很老的版本，笔者的openstack是pike版，所以需要做一些修改
修改如下：
- 1./usr/share/openstack-dashboard/openstack_dashboard/settings.py文件330行修改
```
'images_panel': False,
```
- 2./usr/share/openstack-dashboard/openstack_dashboard/dashboards/project/images/images/views.py最后的download_image函数修改如下:
```
def download_image(request, **kwargs):
    try:
        image_data = api.glance.image_data(request, kwargs['image_id'])
        image = api.glance.image_get(request, kwargs['image_id'])
        image_format = getattr(image, 'disk_format', 'data')
        response = http.StreamingHttpResponse(
            image_data,
            content_type='application/octet-stream')
        response['Content-Disposition'] = ('attachment; filename="%s.%s"' %
                                           (kwargs['image_id'], image_format))
        response['Content-Length'] = image.size
        return response
    except Exception:
        msg = _('An error occured during download of image.')
        url = reverse('horizon:project:images:index')
        exceptions.handle(request, msg, redirect=url)
```
- 3./usr/share/openstack-dashboard/openstack_dashboard/api/glance.py247行修改如下
```
@profiler.trace
def image_data(request, image_id):
    """Returns an Image object populated with metadata for a given image."""
    return glanceclient(request).images.data(image_id)
```
- 4./usr/share/openstack-dashboard/openstack_dashboard/locale/zh_CN/LC_MESSAGES下面的mo对应的文件需要增加
```
msgid "Edit Image"
msgstr "编辑镜像"

msgid "Download Image"
msgstr "下载镜像"
```
ok,重启一下httpd或者apache2服务
