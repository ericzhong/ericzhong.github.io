---
layout: post
title: 上传头像
tags: web
category: it
---

用户在网站注册后一般都会需要一个头像，如果是用户自己上传图片，需要提供裁剪图片的工具，然后按照网站的显示比例（比如100*100）调整图片尺寸（主要是为了节省存储空间以及加快页面响应速度）；

一般实现流程如下：

  1. 用户点击按钮或相框，弹出文件选择对话框
  2. 上传图片，返回图片链接
  3. 弹出图片裁剪窗口
  4. 上传裁剪区域坐标和图片ID，对原图进行裁剪并保存，返回新图片链接
  6. 刷新图片

&nbsp;

## 代码分析

注意：有些前端代码需要放到`ready`里面，有些要放全局（省略）

    jQuery(document).ready(function(){
        ...
    });

###点击图片，弹出选择文件对话框

这里使用点击图片后用程序触发弹框的方式；

> 网上有说IE不支持`<input type='file'>`的程序触发(尚未测试)，必须用手点，所以普遍[方案](http://www.quirksmode.org/dom/inputfile.html)是将Input变成透明，然后覆盖在触发控件的上方，感觉是点击图片，实际是点击了Input；

img在用户设置页面表现为一个相框，显示默认图片，这里省略了上下文；  
form表单是隐藏的，由程序提交，返回的数据加载到同样隐藏的iframe里面，这样页面就不会刷新；

    jQuery('#avatar_img').click(function(click){
        jQuery('#avatar_input').click();
    });

    <form id="avatar_form" action="/cloud/overview/profile/avatar" enctype="multipart/form-data" style="display:none;" method="POST" target="avatar_iframe">
        <input id="avatar_input" name="picture" type="file" class="file" accept="image/*" />
    </form>
    <img id="avatar_img" src="/cloud/static/images/profilethumb.png" alt="" class="img-polaroid" />
    <iframe id='avatar_iframe' name="avatar_iframe" style="display: none;"></iframe>

###上传图片

用户选择图片并保存后，Input的内容会改变，因此可以监听change事件来自动提交表单；

    jQuery('#avatar_input').change(function(click){
        jQuery('#avatar_form').submit();
    });

### 保存图片，返回链接

这段是Server端的代码，用来处理之前提交的表单（使用基于Python的Flask框架，不过使用其它语言或框架逻辑不变）

将图片保存在cache目录里，文件名称为一个随机串，必须加上扩展名，否则后面用PIL裁剪时会报错；

最后返回一个脚本，返回的页面被加载到iframe中，当iframe页面加载完毕时会执行指定程序（iframe_onload），参数就是图片的URL；

    import os
    from datetime import datetime
    import uuid
    import imghdr

    # !!! URL can't end with "/"
    @expose('/overview/profile/avatar', methods=('GET','POST'))
    def avatar(self):
        return_url = url_for(".profile")

        if flask_request.method == 'POST':
            file = flask_request.files['picture']

            if file:
                dir = os.path.join(current_app.root_path, 'static', 'cache')

                # create tmp dir
                if not os.path.exists(dir):
                    try:
                        os.makedirs(dir)
                    except OSError as exc: # Python >2.5
                        if exc.errno == errno.EEXIST and os.path.isdir(dir):
                            pass
                        else:
                            raise

                # get unique filename
                while True:
                    filename = datetime.now().strftime('%Y%m%d%H%M%S') + uuid.uuid4().hex
                    path = "%s/%s" % (dir, filename) 
                    if not os.path.exists(path):
                        break

                try:
                    file.save(path)
                    ext = imghdr.what(path)
                    new_path = "%s.%s" % (path, ext)
                    os.rename(path, new_path)
                except:
                    print "------------------Error: save or open %s" % path 
                    return redirect(return_url)

            else:
                return redirect(return_url)

            src = url_for('.static', filename='cache/%s' % os.path.basename(new_path)) 

            return '''<script type="text/javascript">
                          window.onload=function(){
                              window.parent.iframe_onload("%s");
                          }
                      </script>''' % src
        return redirect(return_url)

### 弹出图片裁剪窗口

`iframe_onload`会弹出图片裁剪对话框，使用的是`bootstrap`的模态框，默认隐藏；  
弹框中有一个图片选择区和一个图片预览区，使用插件[imgAreaSelect](http://odyniec.net/projects/imgareaselect/)

选择区的图片加载完成后需要调整一下尺寸，将图片塞满整个区域，但是要按比例缩放；(注释掉的两行代码可使选择区域去除空闲部分)  
调整尺寸需要设置`max-height/max-width : none`，因为bootstrap的影响；

预览区的原理就是计算选择框尺寸相对预览区尺寸的放大/缩小倍数，然后将原图按照比例缩放，调整显示位置即可；  
预览框有两个div，外面的用来做相框效果，里面的用来显示图片的指定位置（overflow:hidden）；

`onSelectChange`表示鼠标拖动，可能是缩放选择区域，也可能是拖动已有的选择框；  
`onSelectEnd`表示鼠标弹起，即选择结束，保存坐标到全局变量，后面保存时要用；  

    var select_region = [];
    var origin_width = 0;
    var origin_height = 0;

    function save(img, selection) {
        select_region = [selection.x1, selection.y1, selection.x2, selection.y2];
    }

    function preview(img, selection) {
        if (!selection.width || !selection.height)
            return;
    
        // preview size: 100*100
        var scaleX = 100 / selection.width;
        var scaleY = 100 / selection.height;
    
        var w = img.width;
        var h = img.height;
    
        jQuery('#avatar_preview').css({
            width: Math.round(scaleX * w),
            height: Math.round(scaleY * h),
            marginLeft: -Math.round(scaleX * selection.x1),
            marginTop: -Math.round(scaleY * selection.y1)
        });
    
    } 

    function iframe_onload(src){
    
        var avatar_select = jQuery('#avatar_select');
        var avatar_preview = jQuery('#avatar_preview');
    
        avatar_select.attr('src', src);
        avatar_preview.attr('src', src);
    
        avatar_select.imgAreaSelect({
            aspectRatio: '1:1',
            handles: true,
            fadeSpeed: 200,
            onSelectChange: preview, 
            onSelectEnd: save 
        });
    
        jQuery("#avatar_modal").modal('show');
    };

    // resize img
    jQuery('#avatar_select').load(function(){
        var width = this.width; 
        var height = this.height;

        origin_width = width;
        origin_height = height;

        var parent = jQuery(this).parent();
        var max_width = parent.width();
        var max_height = parent.height();

        if(width/height > max_width/max_height){
            jQuery(this).width(max_width);
            //parent.height(jQuery(this).height());
        }else{
            jQuery(this).height(max_height);
            //parent.width(jQuery(this).width());
        };
    });


    <div class="modal hide fade" id="avatar_modal">
        <div class="modal-header">
            <a class="close" data-dismiss="modal">×</a>
            <h3>Upload Photo</h3>
        </div>
        <div class="modal-body">
            <div style="float: left; width: 60%;">
                <p style="font-size:150%;">Select Area</p>
                <div class="img-polaroid" style="width: 300px; height: 300px;">
                    <img id="avatar_select" />
                </div>
            </div>
            <div style="float: left; width: 40%;">
                <p style="font-size:150%;">Preview</p>
                <div class="img-polaroid" style="width: 100px; height: 100px;">
                    <div style="width: 100px; height: 100px; overflow: hidden;">
                        <img id="avatar_preview" style="max-height:none; max-width:none" />
                    </div>
                </div>
            </div>
        </div>
        <div class="modal-footer">
            <a id="upload_avatar" href="#" class="btn btn-primary">Upload</a>
            <a href="#" class="btn" data-dismiss="modal">Cancel</a>
        </div>
    </div>

### 上传裁剪区域坐标和图片ID

当用户确认后，可以按照之前的方法，通过选择区图片与原图的比例计算得到实际的区域坐标；  

    function destroy_avatar_dialog() {
        select_region = [];
        origin_width = 0;
        origin_height = 0;
    
        jQuery('#avatar_select,#avatar_preview').css('width','').css('height','');
        jQuery('#avatar_select').imgAreaSelect({
            hide: true
        });
    }

    jQuery("#avatar_modal").on("hide", function () {
        destroy_avatar_dialog();
    });

    jQuery("#upload_avatar").click(function() {
        var img = jQuery('#avatar_select');

        var scaleX = origin_width/img.width();
        var scaleY = origin_height/img.height();
        var origin_region = [Math.round(select_region[0] * scaleX),
                             Math.round(select_region[1] * scaleY),
                             Math.round(select_region[2] * scaleX),
                             Math.round(select_region[3] * scaleY)];

        jQuery("#avatar_modal").modal('hide');

        // basename of src      
        name = (jQuery('#avatar_select').attr('src')).split(/[\\/]/).pop();
        var data = {"x1":origin_region[0],
                    "y1":origin_region[1],
                    "x2":origin_region[2],
                    "y2":origin_region[3],
                    "name":name};

        jQuery.post( "{{ url_for(".crop") }}", data, crop_end, "json");
    });

### 裁剪图片并保存，返回图片链接

使用PIL（Python Image Library）裁剪图片，然后保存为100*100的尺寸；

因为使用AJAX和JSON格式通信，因此返回也必须是JSON格式，否则jQuery会报错（这里使用Flask的jsonify方法转换）

这里仅仅将图片保存到cache目录里，然后返回图片链接，页面刷新后就失效，因此将图片和用户对应起来还需要用数据库存储必要信息（省略）；

    import Image
    from flask import Flask, redirect, url_for, current_app, jsonify

    @expose('/overview/profile/crop', methods=('GET', 'POST'))
    def crop(self):
        x1 = flask_request.form.get('x1', 0, type=int)
        y1 = flask_request.form.get('y1', 0, type=int)
        x2 = flask_request.form.get('x2', 0, type=int)
        y2 = flask_request.form.get('y2', 0, type=int)
        name = flask_request.form.get('name', None)

        try:
            dir = os.path.join(current_app.root_path, 'static', 'cache')

            # get unique filename
            while True:
                filename = datetime.now().strftime('%Y%m%d%H%M%S') + uuid.uuid4().hex
                path = "%s/%s" % (dir, filename) 
                if not os.path.exists(path):
                    break

            old_path =  "%s/%s" % (dir,name)
            ext = os.path.splitext(old_path)[1]
            new_name = "%s%s" % (filename, ext)
            new_path =  "%s/%s" % (dir,new_name)

            img = Image.open(old_path) 
            box = (x1,y1,x2,y2)
            region = img.crop(box).resize((100,100))
            region.save(new_path)
            os.remove(old_path)
    
            src = url_for('.static', filename='cache/%s' % new_name) 
            ret = {u'src': src}

        except Exception:
            exceptions.handle(request, ignore=True)

        return jsonify(ret)

### 刷新图片

拿到（裁剪后）新的图片URL，刷新图片

    var crop_end = function(data){
        jQuery('#avatar_img').attr('src', data.src);
    } 


