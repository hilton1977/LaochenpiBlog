---
title: Python验证码识别
date: 2018-05-25 16:27:08
tags:
    - python
    - 技术
---

>Python 第三方库 PIL Pytesseract tesseract-ocr  进行爬虫验证码识别

## 1.Python 第三方库依赖
   通过cmd控制台进入python pip目录执行pip install requests 进行安装
   其他的第三方库都可以通过这种形式进行安装
```bash
#进入Python脚本文件夹
cd C:\Users\serwer\AppData\Local\Programs\Python\Python36-32\Scripts
#安装 requests 请求http库
pip install requests 
#安装 pytesseract 基础识别库
pip install pytesseract
#安装 Image图片处理 为更好识别验证码
pip install Image
#显示requests相关信息
pip show requests
```
   可以通过配置pip环境变量达到在任意文件夹目录进行pip脚本执行

## 2.OCR图形识别软件 （Google维护的开源的OCR）
Tesseract-ocr [github地址](https://github.com/tesseract-ocr/tesseract) window 可选择[Tesseract-ocr-setup-3.05.01.exe](https://digi.bib.uni-mannheim.de/tesseract/tesseract-ocr-setup-4.00.00dev.exe)

``` pthyon
    import requests
    import pytesseract
    from PIL import Image
    
    imagePath = "D:\\1.gif"
    imageUrl = "http://112.112.9.205:88/ValiateNum.ashx"
    
    def getAuthCodeImage():
        r = requests.get(imageUrl， stream=True)
        with open(imagePath， 'wb') as fd:
    	    for chunk in r.iter_content(1024):
            	fd.write(chunk)
    	        fd.close
    
    def disposeImage():
    	image = Image.open(imagePath)
    	table = []    
    	for i in range(256):    
    	    if i < 140:    
    	        table.append(0)    
    	    else:    
    	        table.append(1)  
    	image = image.convert('L')
    	image = image.resize((300，100)，Image.BILINEAR)
    	image = image.point(table，'1')  
    	image.save("D:\\1.png"，"png")
    
    
    def discernCode():
    	im=Image.open("D:\\1.png")
    	code = pytesseract.image_to_string(im)
    	print(code)
    
    #获取验证码并保存
    getAuthCodeImage()
    #验证码图片处理 灰阶处理
    disposeImage()
    #识别验证码
    discernCode()
```


