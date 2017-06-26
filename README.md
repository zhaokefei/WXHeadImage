# 基于微信好友图像合成自己的微信图像

## 使用的库

- itchat (登录微信获取数据)
- pillow (处理合成图片）

可以通过 `pip install -r requirements.txt`来安装依赖（建议创建虚拟环境安装)

## 实现过程

1. 登录微信
2. 获取好友信息并将图片存入到本地指定目录下
3. 通过pillow库来将图片合成为一个完整的图片

### 获取好友头像
*itchat提供了非常方便的操作微信的api。*

以下为登录获取最简单的文本消息的代码：
```
import itchat

@itchat.msg_register(itchat.content.TEXT)
def print_content(msg):
    print(msg['Text'])

itchat.auto_login()
itchat.run()
```

调用`auto_login`便可获取登录所需的二维码，微信扫码即实现了登录；通过`@itchat.msg_register()`装饰器注册`print_content`方法，使其成为处理文本消息的函数。以上便是itchat最基本的功能。

目前我们需要的是通过调用获取好友的api获取好友，然后在通过获取好友的头像信息来取得好友头像，接着存入数据库中。
需要用到的API：
- 获取好友列表： `itchat.get_friends()`
- 获取好友头像： `itchat.get_head_img(userName=username)`

调用以上API即可完成好友头像获取。以下为完整代码：
```
import os
import re
import itchat


# 获取好友信息并取出图片
def get_friend_imgs(save_path, get_img_nums=100):
    # 创建存储图片路径
    if os.path.exists(save_path):
        save_path = input(u'该路径已存在，请输入其他目录路径: ')
    os.mkdir(save_path)
    # 获取好友列表
    friends = itchat.get_friends()
    if get_img_nums > len(friends):
        get_img_nums = len(friends)
        print(u'需要获取的图片数量大于好友数量，取好友数量： %s' % len(friends))
    for num, friend in enumerate(friends):
        friend_name = friend['NickName'] or friend['UserName']
        # 好友名字中带空格的用下划线替代
        friend_name = re.sub(r'[\s+]', '_', friend_name)
        friend_img = itchat.get_head_img(userName=friend['UserName'])
        with open(save_path + '/' + friend_name + str(num+1).zfill(3) + '.jpg', 'wb') as f:
            print(u'正在写入 %s 的图像, 还要写入 %s 个' % (friend_name, get_img_nums-num))
            f.write(friend_img)
        if num > get_img_nums:
            print(u'%s 个图片写入完毕' % get_img_nums)
            break
```

### 合成图片

获取完好友头像并存储下来后，便是合成头像，此时该强大的图像处理库pillow登场。
在使用pillow库之前我们需要了解到:
- 微信的头像采用的像素为(640, 640);所以我们最终合成的头像尺寸也为(640, 640)
- 基于上一点，需要对每一个将要嵌入的图片做处理，根据每行放的图片数进行缩小
- 图片从左往右从上往下依次合成排布，需要定义每一个图片在最终生成的图片中的位置

使用 `thum_im=im.resize()`方法来缩小图片尺寸。其中 `im = Image.open('file1.jpg')`对象。

使用 `toImage.paste(thum_im, (x, y))`方法来粘贴每一张图像至新生成图片对应的位置中。其中`toImage = Image.new('RGBA', (640, 640))`为新创建的图片;`(x, y)`为`thum_im`图片粘贴至新生成图片中对应的位置。

以下为合成图片的代码：
```
from PIL import Image

def generate_image(path, gen_filename='multi_img', row_num=10):
    """
    @param path: 需要合并图片集的位置
    @param gen_filename: 生成的图片的名字
    @param row_num: 一行放置的图片数
    """
    # 确定文件中可用的拼接图片数量
    images = os.listdir(path)
    # 图片缩略尺寸
    slide_size = int(640/row_num)
    thum_size = (slide_size, slide_size)
    # 创建画布
    toImage = Image.new('RGBA', (640, 640))
    # 创建画布游标
    x = 0; y = 0
    # 未写入的图片
    invilid_imgs = []
    for num, img in enumerate(images):
        if img.endswith('.jpg'):
            print(u'写入第 {} 个图片; 图片名为: {}'.format(num, img))
            img = path + '/' + img
            try:
                im = Image.open(img)
            except OSError:
                print(u'%s 未写入' % img)
                invilid_imgs.append(img)
                continue
            # 每行放`row_num`数量图片后，每个图片的尺寸
            if im.size != thum_size:
                thum_im = im.resize(thum_size, Image.ANTIALIAS)
            else:
                thum_im = im
            print('x: {}; y: {}'.format(x * slide_size, y * slide_size))
            toImage.paste(thum_im, (x * slide_size, y * slide_size))
            x += 1
            if x == row_num:
                x = 0
                y += 1
    print(u'未写入的图片有: {}'.format(' @|@ '.join(invilid_imgs)))
    # 保存图片
    toImage.save(path + '/' + gen_filename + '.jpg')
    print(u'生成的文件位于: {}; 名为: {}'.format(path, gen_filename + '.jpg'))
```

### 启动微信并调用以上方法

```
if __name__ == '__main__':
    itchat.auto_login()
    # 存放图片集的文件夹
    path = 'friend_kefei'
    get_friend_imgs(path)
    generate_image(path)
```

完整代码参见 [合成微信图像](https://github.com/zhaokefei/WeChatHeadImage)
