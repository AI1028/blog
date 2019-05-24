@[toc](OCR项目文档)
# OCR使用说明
## 接口能力
**通用文字识别**：对文本文件进行识别，返回给用户文字

## 请求说明
**HTTP方法**：
`post`
**请求URL**：
`http：//---.---.---,---:8081/ocrstart`
**url参数:**
data参数：图像数据，base64编码后的图像数据直接放入到data中
## 请求代码示例：
**python版本：**
```python
import requests
import base64
import json

with open("test.png","rb") as f:
	base64_data = base64.b64encode(f.read())
return_data = requests.post("http://---.---.----.---:8081/ocrstart",data=base64_data)
```

## 返回数据格式
`requests.models.Response`

## 返回数据说明
采用json.loads(return_data.text)转化为json数据，json的数据格式如下
```json
[
  {
    "line" : ********文字所在行数********,
    "data" : *********返回的文字*********,
    "loc"  : ********文字框的位置********
  },

    .
    .
    .
  {
    "line" : ********文字所在行数********,
    "data" : *********返回的文字*********,
    "loc"  : ********文字框的位置********
  }，

  {  
    "picture" : *******返回图片的base64编码********
  }
]
```
**说明：**
返回数据处理后为列表类型，列表的每一项的内容均为字典，除最后一项中存放标注了待识别文字的位置的图片的base64编码，列表的其他项均是识别结果信息，其中`line`表示文字处于第几行；`data`表示识别的文字内容；`loc`表示每行文字的位置信息

## 返回示例
**传入图片：**
![示例图片](https://upload-images.jianshu.io/upload_images/14576254-da59df65cdbaaebd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)

```
[
{
"data":"施与受的定律就是能量守恒的定律。",
"line":1,
"loc":"[ 072384700105384104 ]"
},
{
"data":"就是说: 你布施出去的任何东西，终将成倍地回报到你身上。",
"line":2,
"loc":"[ 01476401470179640178 ]"
},
{
"data":"你布施金钱或物质",
"line":3,
"loc":"[ 02222242230252224253 ]"
},
{
"data":"将会成倍地获得金钱或物质回报;",
"line":4,
"loc":"[ 02973682970330368330 ]"
},
{
"data":"你布施欢喜，让他人愉悦，",
"line":5,
"loc":"[ 03722883720404288404 ]"
},
{
"data":"将会成倍地得到他人回报给你的欢喜",
"line":6,
"loc":"[ 04474004490482400484 ]"
},
{
"data":"你布施安定，让他人心安",
"line":7,
"loc":"[ 05212885220554288555 ]"
},
{
"data":"将会成倍地得到安乐;",
"line":8,
"loc":"[ 05972405980632240632 ]"
},
{
"data":"相反，如果你施加于别人的是不安、憎恨、怒气、忧愁,",
"line":9,
"loc":"[ 06755926730706592704 ]"
},
{
"data":"也将成倍地得到这些报应。",
"line":10,
"loc":"[ 07482887490779288780 ]"
},
{
"data":"宇宙是一台无比精密的精算器。你给予出去的任何东西，最后都会以各种形式回到你的",
"line":11,
"loc":"[ 08239128210858912856 ]"
},
{
"data":"身边。",
"line":12,
"loc":"[ 086180861089280893 ]"
},
{
"picture":"data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA0gAAAMpCAIAAACxLYRzAAEAAElEQVR4nOzdd5gcx30n/F91nhx3Z3......"
}
]
```
**传回图片：**
![返回图片](https://upload-images.jianshu.io/upload_images/14576254-c1f7c55ad3ca0638.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)

# OCR项目主要代码
## demo.py
```python 
def sort_box(box):
    """ 
    对box进行排序
    """
    box = sorted(box, key=lambda x: sum([x[1], x[3], x[5], x[7]]))
    return box

def dumpRotateImage(img, degree, pt1, pt2, pt3, pt4):
    height, width = img.shape[:2]
    heightNew = int(width * fabs(sin(radians(degree))) + height * fabs(cos(radians(degree))))
    widthNew = int(height * fabs(sin(radians(degree))) + width * fabs(cos(radians(degree))))
    matRotation = cv2.getRotationMatrix2D((width // 2, height // 2), degree, 1)
    matRotation[0, 2] += (widthNew - width) // 2
    matRotation[1, 2] += (heightNew - height) // 2
    imgRotation = cv2.warpAffine(img, matRotation, (widthNew, heightNew), borderValue=(255, 255, 255))
    pt1 = list(pt1)
    pt3 = list(pt3)

    [[pt1[0]], [pt1[1]]] = np.dot(matRotation, np.array([[pt1[0]], [pt1[1]], [1]]))
    [[pt3[0]], [pt3[1]]] = np.dot(matRotation, np.array([[pt3[0]], [pt3[1]], [1]]))
    ydim, xdim = imgRotation.shape[:2]
    imgOut = imgRotation[max(1, int(pt1[1])) : min(ydim - 1, int(pt3[1])), max(1, int(pt1[0])) : min(xdim - 1, int(pt3[0]))]

    return imgOut

def charRec(img, text_recs, adjust=False):
   """
   加载OCR模型，进行字符识别
   """
   results = {}
   xDim, yDim = img.shape[1], img.shape[0]
    
   for index, rec in enumerate(text_recs):
       xlength = int((rec[6] - rec[0]) * 0.1)
       ylength = int((rec[7] - rec[1]) * 0.2)
       if adjust:
           pt1 = (max(1, rec[0] - xlength), max(1, rec[1] - ylength))
           pt2 = (rec[2], rec[3])
           pt3 = (min(rec[6] + xlength, xDim - 2), min(yDim - 2, rec[7] + ylength))
           pt4 = (rec[4], rec[5])
       else:
           pt1 = (max(1, rec[0]), max(1, rec[1]))
           pt2 = (rec[2], rec[3])
           pt3 = (min(rec[6], xDim - 2), min(yDim - 2, rec[7]))
           pt4 = (rec[4], rec[5])
        
       degree = degrees(atan2(pt2[1] - pt1[1], pt2[0] - pt1[0]))  # 图像倾斜角度

       partImg = dumpRotateImage(img, degree, pt1, pt2, pt3, pt4)

       if partImg.shape[0] < 1 or partImg.shape[1] < 1 or partImg.shape[0] > partImg.shape[1]:  # 过滤异常图片
           continue

       image = Image.fromarray(partImg).convert('L')
       text = keras_densenet(image)
       
       if len(text) > 0:
           results[index] = [rec]
           results[index].append(text)  # 识别文字
 
   return results

def model(img, adjust=False):
    """
    @img: 图片
    @adjust: 是否调整文字识别结果
    """
    cfg_from_file('./ctpn/ctpn/text.yml')
    text_recs, img_framed, img = text_detect(img)
    text_recs = sort_box(text_recs)
    result = charRec(img, text_recs, adjust)
    return result, img_framed
```
## server.py:服务器端处理数据返回规定结果
```python
def server(img_b64encode):# 服务器端调用模型处理返回结果
    img_b64decode = base64.b64decode(img_b64encode)
    image_file = io.BytesIO(img_b64decode)
    #img = Image.open(image_file).show
    image = np.array(Image.open(image_file).convert('RGB'))
    t = time.time()
    result, image_framed = ocr.model(image)
    print("Mission complete, it took {:.3f}s".format(time.time() - t))
    result_dir = './test_result'
    if os.path.exists(result_dir):
        shutil.rmtree(result_dir)
    os.mkdir(result_dir)
    output_file = os.path.join(result_dir,"temp.png")
    Image.fromarray(image_framed).save(output_file)
    with open(output_file,"rb") as f:
        bs_data = base64.b64encode(f.read())
    info = []
    for key in result:
        temp = {}
        temp["line"] = key
        temp["loc"] = str(result[key][0])
        temp["data"] = result[key][1]
        info.append(temp)
    bs = {}
    #bs["picture"] = bs_data.decode("gb2312")
    info.append(bs)
    return info
```

## resetful.py：使用flask提供接口
```python
ocrapi = Flask(__name__)

@ocrapi.route('/ocrstart', methods=['POST','GET'])
def get_result():
    import demo
    result = demo.server(request.data)
    return json.dumps(result, ensure_ascii = False)
    #t = {}
    #t["data"]= result
    #return json.dumps(t, ensure_ascii = False)
    #img_b64encode = request.data
    #img_b64decode = base64.b64decode(img_b64encode)
    #image = io.BytesIO(img_b64decode)
    #img = Image.open(image)
    
if __name__ == "__main__":
    ocrapi.run("0.0.0.0", port=8081,threaded=True)
```
## 访问服务器代码示例
```python
import time
import requests
import base64
import json

with open("demo.png","rb") as f:
    base64_data = base64.b64encode(f.read())      #open the picture you want to ocr

result = requests.post("http://192.168.3.126:8081/ocrstart",data=base64_data)
print(type(result))
json_data = json.loads(result.text)
for line in json_data[:-1]:
    print("line:\t"+str(line["line"]))
    print("data:\t"+line["data"])
    print("loc:\t"+line["loc"])
    print("=======================")

img_bs64 = json_data[-1]["picture"].encode(encoding="utf-8")
img_data = base64.b64decode(img_bs64)
with open('001.png', 'wb') as f:
      f.write(img_data)

time.sleep(30)
```

# OCR 网页页面
## 网页实现主要代码
```js
function changeBackground(parm){
    /*var path = window.URL.createObjectURL(imgFile.files[0]);*/
    var canvas = document.getElementById("tech-canvas-container");
    canvas.style.background =  "url(" + parm + ")";
    canvas.style.backgroundSize = "100% 100%";//"50%";
};

function gallary(num) {
    var imgList = document.getElementsByClassName("image-select-item");
    for(var i=0; i<5; i++){
        if(i != num)
            imgList[i].style.opacity = 0.5;
        else
            imgList[i].style.opacity = 1;
    }

    path = "pictures/" + String(num) + ".jpg";
    changeBackground(path);
    var img = new Image();
    img.src = path;
    console.log(img.decode())

    document.getElementById("content").innerHTML = 'detecting ...';
    ajax({
        url:'http://172.16.60.62:7000',
        type:'POST',
        dataType:'json',
        data:{image:img.src.substring(img.src.indexOf(',')+1)},
        success:function(response,xml){
            //请求成功后执行的代码
            var result = eval("("+response+")");
            imgg = "data:image/jpeg;base64,"+result.image;
            changeBackground(imgg);
            document.getElementById("content").innerHTML = response;
        },
        error:function(status){
            console.log('failed')
        }
    });
};


function upLoad(imgFile) {
    var file = imgFile.files[0];
    var path = window.URL.createObjectURL(file);
    var reader = new FileReader();
    reader.readAsDataURL(file);
    reader.onload = function(e) {
        changeBackground(this.result);
        document.getElementById("content").innerHTML = 'detecting ...';
        //调用ajax函数
        ajax({
			url:'http://192.168.2.114:8081/ocrstart',
            type:'POST',
            dataType:'json',
            data:{"image":this.result.substring(this.result.indexOf(',')+1)},
            success:function(response,xml){
                var result = response;
                var object = eval("("+result+")");
                var imgg = object.res;
		document.getElementById("content").innerHTML = JSON.stringify(imgg);
              
            },
            error:function(status){
                console.log('failed'+status)
            }
        });
    };
};
function ajax(options){
    options=options||{};
    options.dataType=options.dataType||'json';
    params = formParams(options.data);
    var xhr;
    if(window.XMLHttpRequest){
        xhr=new XMLHttpRequest();
    }else{
        xhr=ActiveXObject('Microsoft.XMLHTTP');
    }

    //连接和发送-第二步
    if(options.type=='GET'){
    }else if(options.type=='POST'){
        xhr.open('POST',options.url,true);
        xhr.onreadystatechange=function(){
            if(xhr.readyState==4){
                var status=xhr.status;
                if(status>=200&&status<300){
                    options.success&&options.success(xhr.responseText,xhr.responseXML);
                }else{
                    options.error&&options.error(status);
                }
            }
        }
        xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
        xhr.send(params);
    }
};

//格式化参数
function formParams(data){
    var arr = [];
    for(var name in data){
        arr.push(encodeURIComponent(name) + '=' + encodeURIComponent(data[name]));
    }
    return arr;
};
```
## OCR WEB效果展示
**网页效果:**
![图片1.png](https://upload-images.jianshu.io/upload_images/14576254-e581ef28e81e461b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**展示结果：**
![图片2.png](https://upload-images.jianshu.io/upload_images/14576254-645369ec6607d367.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

