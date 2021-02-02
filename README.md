# MC_OCR - TOP 2 solution cho bài toán trích xuất thông tin hóa đơn


## CÀI ĐẶT
**Cài đặt source code và thư viện**

```
cd mc_ocr
pip3 install --upgrade pip
pip3 install -r requirements.txt
pip3 install -e .
```
```
cd text_detector/PaddleOCR
pip3 install -e .
python3 -m pip install paddlepaddle==2.0rc1 -i https://mirror.baidu.com/pypi/simple
```

```
cd text_classifier/vietocr
pip3 install -e .
```

## TASK 1: Đánh giá chất lượng hóa đơn
**Trainning:**
Chuẩn bị dữ liệu training giống như file *image_quality_evaluation/train.txt* và chạy

```
cd image_quality_evaluation
python3 train.py --train_mode=1 --path_file=train.txt  --img_size=640
```
Model đạt kết quả cao nhất của team mình đạt được tại epoch 170

**Evaluate:**

Tạo file *image_quality_evaluation/private_test.txt* và chạy

```
cd image_quality_evaluation
python3 train.py --train_mode=0 --path_file=private_test.txt --img_size=640 --path_pretrain=path_to_model --path_save_result=result.txt
```
Kết qủa cuối cùng sẽ được lưu trong file **result.txt**

## TASK 2: Trích xuất thông tin quan trọng từ hóa đơn

Dươí đây là các bước chuẩn bị dữ liệu và huấn luyện.
Đầu tiên, các bạn hãy tải dataset của BTC về và tạo symbolic links
```
cd mc_ocr/data
ln -s [your_downloaded_train_image_folder] mc_ocr_train
ln -s [your_downloaded_private_test_image_folder] mc_ocr_private_test
```

### EDA - Phân tích dữ liệu
Tập dữ liệu training do BTC đưa có có tổng cộng 1155 ảnh. 
Tuy nhiên team mình thấy nhiều ảnh trong số đó chưa đạt yêu cầu như gắn nhãn sai, nhầm thông tin... nên mình có dùng 1 số rules để lọc chúng đi.

```
cd EDA
python3 filter_training_data_by_rules.py
```
Dữ liệu training sau khi lọc sẽ còn 1090 ảnh.

Tiếp theo bên mình có lọc ra danh sách tên cửa hàng + địa chỉ từ dữ liệu training.
```
cd EDA
python3 get_store_dict.py
```
Kết quả sã là file *final_data.json* chứa danh sách các cửa hàng (tên và địa chỉ). 
Danh sách này sẽ được dùng để sửa lỗi cho phần OCR


### TRAINING
Hệ thống của team mình có 5 bước chính giống như hình vẽ dưới đây:

![AICR](http://107.120.70.222:6969/aicr/mc_ocr/-/blob/release/pipeline_task_2.JPG "AICR")

Trước hết, các bạn hãy sửa biến sau trong file *config.py*
```
dataset = 'mc_ocr_train_filtered'
```

#### 1. text_detector 
Bước này sẽ tìm vị trí của vùng chữ trên ảnh. Team mình sử dụng pre-trained từ [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) mà không finetune lại gì cả. 
Các bạn hãy download pre-trained từ link [này](https://paddleocr.bj.bcebos.com/dygraph_v2.0/ch/ch_ppocr_server_v2.0_det_infer.tar), 
giải nén và chỉnh sửa đường dẫn đến pre-trained trong file *config.py*
```buildoutcfg
det_model_dir = [your extracted folder]
```
#### 2. rotation corrector
Bước này sẽ làm nhiệm vụ xoay lại hóa đơn cho thẳng. Bên mình có tạo dữ liệu và training từ đầu cho bước này. Feature extractor được sử dụng là Mobilenetv3

**text detector**
```
cd text_detector/PaddleOCR
python3 tools/infer/predict_det.py

```

1. lọc các ảnh bị ngược hoặc xoay ngang trong tập train:
    + sử dụng confidence của text classify để lọc, với threshold là 0.7
    ```
    python3 process_mc_ocr_data.py
    ```
2. tạo synthetic data và augmentation real data từ dữ liệu ở trên:
    ```
    python3 data_process.py
    ```
3. training:
    sửa line 10, 11 file *rotation_corrector/experiments/mobilenetv3_filtered_public_train.yaml* theo đường dẫn từ bước 2. (base_output_dir)
    ```
    python3 train_config.py --cfg experiments/mobilenetv3_filtered_public_train.yaml
    ```
    
Pre-trained team mình sử dụng đã để sẵn trong *rotation_corrector/weights*

#### 3. textline rotation
Sau khi xoay lại hóa đơn sẽ vẫn còn nhiều dòng chữ bị nghiêng, 
bước này sẽ cắt vùng chữ đó và xoay lại cho thẳng để phần OCR được tốt hơn.
Team mình chỉ sử dụng xử lý ảnh cơ bản cho phần này
#### 4. text classifier
Đây chính là bước OCR đọc chữ từ vùng ảnh đã được xoay từ trên. 
Team mình sử dụng pre-trained từ open soucre nổi tiếng [Vietocr](https://github.com/pbcquoc/vietocr). 
Rất cảm ơn anh pbcquoc đã public một model rất tốt ra cộng đồng.
Các bạn hãy download pretrained của mô hình vgg19_seq2seq tại [đây](https://drive.google.com/uc?id=1nTKlEog9YFK74kPyX0qLwCWi60_YHHk4) và chỉnh sửa đường dẫn trong file *config.py*
```buildoutcfg
cls_model_path = [your downloaded model]
```

#### 5. key information extraction
Bước này sử dụng [PICK model](https://github.com/wenwenyu/PICK-pytorch) của tác giả wenwenyu để trích xuất thông tin từ hóa đơn. Team mình có tạo dữ liệu và huấn luyện mô hình từ đầu cho bước này.

Đầu tiên, hãy sửa các biến sau trong file *config.py*: 
```
dataset = 'mc_ocr_train_filtered'
det_visualize = False
rot_visualize = False
cls_visualize = False
```
Sau đó, chạy lần lượt các bước **text detector**, **rotation corrector** và **text classifier**
```
cd text_detector/PaddleOCR
python3 tools/infer/predict_det.py
```

```
cd rotation_corrector
python3 inference.py
```

```
cd text_classifier
python3 pred_ocr.py
```

Dùng model thu được ở bước **2. rotation_corrector** để chỉnh sửa file csv:

```
cd rotation_corrector
python3 rotate_csv.py
```

Sau đó chuẩn bị dữ liệu training

```
cd key_info_extraction
python3 create_train_data.py
```
Bước trên sẽ tạo ra 2 file train.csv và val.csv trong thư mục ... Tiếp theo các bạn hãy chỉnh sửa đường dẫn đến 2 file trên trong file *key_info_extraction/PICK/config.json* (line 59 và 74)

Cuối cùng sẽ là bước training 
 
```
cd key_info_extraction/PICK
bash run.sh
```
Kết quả của bước training sẽ là file *model_best.pth* nằm trong thư mục *key_info_extraction/PICK/saved/models/PICK_Default/test...*
Các bạn hãy chỉnh sửa đường dẫn đến file đó trong *config.py*
```buildoutcfg
kie_model = [your trained model]
```
model mà team mình train được có thể download tại [đây](https://drive.google.com/file/d/1G3jNF2eEANN5B_tN5bHkD09TMZnQi2cD/view?usp=sharing)


### SUBMISSION 

Chuyển dataset sang private test trong file *config.py*

```
dataset = 'mc_ocr_private_test'
```
Sau đó chạy lần lượt các bước sau

**text detector**
```
cd text_detector/PaddleOCR
python3 tools/infer/predict_det.py

```

**rotation corrector**
```
cd rotation_corrector
python3 inference.py

```

**text classifier**
```
cd text_classifier
python3 pred_ocr.py

```

**key information extraction**
```
cd key_info_extraction/PICK
python3 test.py

```
**submission**
```
cd submit
python3 submit.py

```
Kết quả cuối cùng là file *submit/private_test/result.csv*

## TỔNG KẾT 
#### Open source đã sử dụng:
- PaddleOCR: https://github.com/PaddlePaddle/PaddleOCR
- PICK-Pytorch: https://github.com/wenwenyu/PICK-pytorch
- Vietocr: https://github.com/pbcquoc/vietocr

Trong đó team mình sử dụng pre-trained từ PaddleOCR và Vietocr

## NHẬN XÉT 
- Dữ liệu từ BTC khá lớn (tổng cộng 2k ảnh, với 1155 ảnh đã được gán nhãn). 
Tuy nhiên, phàn lớn là hóa đơn từ MINIMART ANAN và Vincommerce nên sự đồng đều và đa dạng là không cao.
- Dữ liệu đã gán nhãn có tỉ lệ lỗi khoảng 5-10%
- Yêu cầu không sử dụng dữ liệu ngoài với không gán nhãn thủ công khiến cho kết quả đạt được không quá tốt.

## TEAM MEMBER
- cuongnd (nd.cuong1@samsung.com)
- anhnt (nt.anh6@samsung.com)
- chungnx (nx.chung@samsung.com)