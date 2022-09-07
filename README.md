# tengine_khadas_sdk
# YOLOv5 на Khadas
## Установка Ubuntu на Khadas VIM3
1. Следуя [инструкциям](https://docs.khadas.com/linux/vim3/index.html) из раздела Quick start устанавливаем Ubuntu. Образ можно взять с [официального сайта](https://docs.khadas.com/linux/firmware/Vim3UbuntuFirmware.html) или брать уже 
протестированную версию [V1.0.7-210625](https://drive.google.com/drive/folders/1FUXloO80ecwliYHQxfVgwSeL5ceh2Hhl?usp=sharing).
## YOLOv5
1. Клонируем форк репозитория, с leakyReLU свёртками и экспортом в ONNX обрезанного слоя Detect
```
git clone https://github.com/SashaAlderson/yolov5
```
2. Обучаем сеть в течение 300 эпох следуя гайдам, либо используем [предобученную](https://drive.google.com/drive/folders/1wlErIkcGLRwXylHBNNuMXS2gnXjkixCY?usp=sharing).
## Конвертация на NPU
1. Клонируем репозиторий с готовыми бинарными файлами для конвертации, конвертировать можно на стороннем устройстве.
```
git clone https://github.com/SashaAlderson/tengine_khadas_sdk
cd tengine_khadas_sdk/tengine_tools
```
2. Устанавливаем opencv
```
sudo apt install libopencv-dev
```
3. Конвертируем модель в tmfile

  Параметры: 
```bash
$ ./tengine_tools/convert_tool/convert_tool -h

---- Tengine Convert Tool ---- 

Version     : v1.0, 15:43:59 Jun 24 2021
Status      : float32
[Convert Tools Info]: optional arguments:
	-h    help            show this help message and exit
	-f    input type      path to input float32 tmfile
	-p    input structure path to the network structure of input model(*.prototxt, *.symbol, *.cfg, *.pdmodel)
	-m    input params    path to the network params of input model(*.caffemodel, *.params, *.weight, *.pb, *.onnx, *.tflite, *.pdiparams)
	-o    output model    path to output fp32 tmfile

[Convert Tools Info]: example arguments:
	./convert_tool -f caffe -p ./mobilenet.prototxt -m ./mobilenet.caffemodel -o ./mobilenet.tmfile
```
```bash
$ ./tengine_tools/convert_tool/convert_tool -f onnx -m models/yolov5m_leaky_352.onnx -o models/yolov5m_leaky_352.tmfile

---- Tengine Convert Tool ---- 

Version     : v1.0, 15:43:59 Jun 24 2021
Status      : float32
Create tengine model file done: models/yolov5m_leaky_352.tmfile
```
4. Создаём папку с изображениями для калибровки. Желательно иметь 500-1000 изображений из вашего датасета.

5. Запускаем процесс квантизации модели

  Параметры:
```bash
$ ./tengine_tools/quant_tool/quant_tool_uint8 -h
[Quant Tools Info]: optional arguments:
	-h    help            show this help message and exit
	-m    input model     path to input float32 tmfile
	-i    image dir       path to calibration images folder
	-f    scale file      path to calibration scale file
	-o    output model    path to output uint8 tmfile
	-a    algorithm       the type of quant algorithm(0:min-max, 1:kl, default is 0)
	-g    size            the size of input image(using the resize the original image,default is 3,224,224)
	-w    mean            value of mean (mean value, default is 104.0,117.0,123.0)
	-s    scale           value of normalize (scale value, default is 1.0,1.0,1.0)
	-b    swapRB          flag which indicates that swap first and last channels in 3-channel image is necessary(0:OFF, 1:ON, default is 1)
	-c    center crop     flag which indicates that center crop process image is necessary(0:OFF, 1:ON, default is 0)
	-y    letter box      the size of letter box process image is necessary([rows, cols], default is [0, 0])
	-k    focus           flag which indicates that focus process image is necessary(maybe using for YOLOv5, 0:OFF, 1:ON, default is 0)
	-t    num thread      count of processing threads(default is 1)

[Quant Tools Info]: example arguments:
	./quant_tool_uint8 -m ./mobilenet_fp32.tmfile -i ./dataset -o ./mobilenet_uint8.tmfile -g 3,224,224 -w 104.007,116.669,122.679 -s 0.017,0.017,0.017

```
```bash
$ ./tengine_tools/quant_tool/quant_tool_uint8 -m models/yolov5m_leaky_352.tmfile -i calibration \
-o models/yolov5m_leaky_352_uint8.tmfile -g 3,352,352 -a 0  -w 0,0,0 \
-s 0.003922,0.003922,0.003922 -c 0 -t 4 -b 1 -y 352,352


---- Tengine Post Training Quantization Tool ---- 

Version     : v1.2, 15:15:47 Jun 22 2021
Status      : uint8, per-layer, asymmetric
Input model : models/yolov5m_leaky_352.tmfile
Output model: models/yolov5m_leaky_352_uint8.tmfile
Calib images: calibration
Scale file  : NULL
Algorithm   : MIN MAX
Dims        : 3 352 352
Mean        : 0.000 0.000 0.000
Scale       : 0.004 0.004 0.004
BGR2RGB     : ON
Center crop : OFF
Letter box  : 352 352
YOLOv5 focus: OFF
Thread num  : 4

[Quant Tools Info]: Step 0, load FP32 tmfile.
[Quant Tools Info]: Step 0, load FP32 tmfile done.
[Quant Tools Info]: Step 0, load calibration image files.
[Quant Tools Info]: Step 0, load calibration image files done, image num is 500.
[Quant Tools Info]: Step 1, find original calibration table.
[Quant Tools Info]: Step 1, images 00500 / 00500
[Quant Tools Info]: Step 1, find original calibration table done, output ./table_minmax.scale
[Quant Tools Info]: Thread 4, image nums 500, total time 632535.31 ms, avg time 1265.07 ms
[Quant Tools Info]: Calibration file is using table_minmax.scale
[Quant Tools Info]: Step 3, load FP32 tmfile once again
[Quant Tools Info]: Step 3, load FP32 tmfile once again done.
[Quant Tools Info]: Step 3, load calibration table file table_minmax.scale.
[Quant Tools Info]: Step 4, optimize the calibration table.
[Quant Tools Info]: Step 4, quantize activation tensor done.
[Quant Tools Info]: Step 5, quantize weight tensor done.
[Quant Tools Info]: Step 6, save Int8 tmfile done, models/yolov5m_leaky_352_uint8.tmfile

---- Tengine Int8 tmfile create success, best wish for your INT8 inference has a low accuracy loss...\(^0^)/ ----

```
## Советы
1. Используйте больше 300 изображений для калибровки
2. Через параметр -a можно выбирать либо MINMAX (a=0) либо KL (a=1). В некоторых случаях KL может показывать результаты хуже.
В этом плане MINMAX калибровка более быстрая и стабильная. Проверяйте точность моделей перед использованием.
3. Изображения для квантизации должны соответствовать тем, на которых была обучена модель.
