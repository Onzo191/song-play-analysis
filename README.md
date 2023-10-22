# Xây dựng Data Lake trên AWS
**Demo datalake Mạng Xã Hội IS353.O12**

**Nguồn tham khảo:**[jpsalado92 - Datalake AWS EMR](https://github.com/jpsalado92/Udacity-DEND_DataLake-AWSEMR).

## Các dịch vụ, công cụ sử dụng
* AWS S3
* AWS EMR
* Spark

## Giới thiệu

### Dự án này là gì?
Hãy tưởng tượng bạn vừa được thuê làm kỹ sư dữ liệu bởi một startup âm nhạc gọi là Sparkify.
Công ty đã phát triển cơ sở người dùng và cơ sở dữ liệu bài hát của họ đến một mức họ muốn chuyển cơ sở dữ liệu của họ thành một Data Lake. Họ đã cho bạn truy cập vào dữ liệu của họ, bao gồm một thư mục chứa nhật ký JSON về hoạt động của người dùng trên ứng dụng, cũng như một thư mục chứa JSON về siêu dữ liệu về các bài hát trong ứng dụng của họ.

Với tư cách là kỹ sư dữ liệu của họ, bạn được giao nhiệm vụ tải dữ liệu này lên S3 và xây dựng một đường ống ETL trích xuất dữ liệu từ nó, xử lý dữ liệu bằng Spark và tải lại nó vào S3 dưới dạng bảng chiều. Điều này sẽ cho phép nhóm phân tích của họ tiếp tục tìm hiểu về các bài hát mà người dùng của họ đang nghe.

### Các dữ liệu sử dụng (Datasets)
Dữ liệu được lưu trữ trong [input data](data/input).

#### Song Dataset
Tập dữ liệu đầu tiên là một phần của dữ liệu thực tế từ [Million Songs Dataset](https://labrosa.ee.columbia.edu/millionsong/).
Mỗi tệp đều ở định dạng JSON và chứa dữ liệu về một bài hát và nghệ sĩ của bài hát đó.

Các tệp này được phân ra dựa theo ba chữ cái đầu tiên của track ID của mỗi bài hát.
Ví dụ, dưới đây là đường dẫn đến hai tệp trong tập dữ liệu này.
```
song_data/A/B/C/TRABCEI128F424C983.json
song_data/A/A/B/TRAABJL12903CDCF1A.json
```
Và dưới đây là một ví dụ về nội dung của một tệp bài hát duy nhất, TRAABJL12903CDCF1A.json.
```
{
   "num_songs": 1,
   "artist_id": "ARJIE2Y1187B994AB7",
   "artist_latitude": null,
   "artist_longitude": null,
   "artist_location": "",
   "artist_name": "Line Renaud",
   "song_id": "SOUPIRU12A6D4FA1E1",s
   "title": "Der Kleine Dompfaff",
   "duration": 152.92036,
   "year": 0
}
```

#### Log Dataset
Tập dữ liệu thứ hai bao gồm các tệp nhật ký ở định dạng JSON được tạo bởi trình mô phỏng sự kiện này
dựa trên các bài hát trong tập dữ liệu ở trên. Chúng mô phỏng các nhật ký hoạt động ứng dụng từ một ứng dụng nghe nhạc tưởng tượng dựa trên cài đặt cấu hình.

Các tệp nhật ký trong tập dữ liệu mà bạn sẽ làm việc đã được sắp xếp theo ngày. Ví dụ, dưới đây là đường dẫn đến hai tệp trong tập dữ liệu này.

```
log_data/2018-11-12-events.json
log_data/2018-11-13-events.json
```

## Các bước triển khai
### Xác định schema cho Song Play Analysis
Để đơn giản hóa truy vấn và cho phép tổng hợp nhanh (fast aggregations), chúng ta sẽ sử dụng **Star Schema** bằng cách sử dụng các tập dữ liệu bài hát và sự kiện.
Những bảng này sẽ bao gồm:

**1 Fact Table**

* **songplays** - các bản ghi trong dữ liệu sự kiện liên quan đến việc phát bài hát, tức là các bản ghi với trang NextSong
    * _songplay_id, start_time, user_id, level, song_id, artist_id, session_id, location, user_agent_

**4 Dimension Tables**

* **users** - người dùng trong ứng dụng
    * _user_id, first_name, last_name, gender, level_
* **songs** - các bài hát trong cơ sở dữ liệu âm nhạc
    * _song_id, title, artist_id, year, duration_
* **artists** - các nghệ sĩ trong cơ sở dữ liệu âm nhạc
    * _artist_id, name, location, latitude, longitude_
* **time** - dấu thời gian của các bản ghi trong sự kiện phát bài hát được phân rã thành các đơn vị cụ thể
    * _start_time, hour, day, week, month, year, weekday_

Khi lưu trữ các bảng trở lại S3, chúng sẽ ở định dạng **parquet**, cho phép thực hiện tính toán với cụm Spark giúp tạo truy vấn nhanh, đây là [output data](data/output) sau khi hoàn thành các bước triển khai.

### Chuẩn bị
* Tạo tài khoản AWS
* Cài đặt AWS CLI
* Cài đặt OpenSSH Client & Server

### Tạo một IAM User cho API
Tạo IAM User và lưu **AWS_ACCESS_KEY_ID** và **AWS_SECRET_ACCESS_KEY** vào [dl.cfg](dl.cfg). Sau đó config AWS CLI với IAM User này.

### Tạo một S3 bucket
**Tạo S3 bucket**
* Tên bucket: **is353-song-play-analysis**
* Region: Asia Pacific (Singapore) ap-southeast-1

Sau khi tạo, ta lưu tên bucket vào [dl.cfg](dl.cfg). Cấp policies cho IAM User ở trên có quyền thao tác trên Bucket này.

### Upload dữ liệu lên S3 bucket
* Download dataset từ [jpsalado92 - Datalake AWS EMR](https://github.com/jpsalado92/Udacity-DEND_DataLake-AWSEMR) giải nén và upload lên S3 qua AWS CLI.
```
aws s3 sync "path-to-folder\song-data" s3:///is353-song-play-analysis/input/song-data
aws s3 sync "path-to-folder\log-data" s3://is353-song-play-analysis/input/log-data
```

### Tạo Amazon EC2 Keypair
Sau khi tạo Keypair, lưu lại key, setup Security cho nó. (Keyname: is353-song-play-analysis-keypair.pem). Nó sẽ được sử dụng để kết nối tối Cluster qua SSH.

### Chọn kiến trúc Data Lake trong AWS
AWS EMR - Spark only.

#### Tổng quan việc thiết lập Spark Cluster
1. AWS S3 lưu datasets (input_data).
2. Thuê Cluster với dịch vụ Elastic Compute Cloud (EC2).
3. Kết nối tới Spark Cluster này trên máy của mình qua SSH.
4. Khi chạy [mã Spark](etl.py), cụm sẽ nạp datasets từ Amazon S3 vào bộ nhớ của cụm, được phân phối trên mỗi máy trong cụm.
5. Cluster sẽ xuất kết quả vào lại S3 dưới dạng parquet.files.


### Khởi chạy Cụm EMR

#### Cấu hình Cluster
* Name: My cluster
* Amazon EMR release: Amazon EMR release
* Application bundle: Spark (Spark 3.4.1, Zeppelin 0.10.1)
* Log destination in Amazon S3: is353-song-play-analysis
* Instance groups: Primary (m5.xlarge), Core (m5.xlarge), Task (m5.xlarge)
* Provisioning configuration: Core size: 1 instance, Task size: 1 instance
* Log destination in Amazon S3: is353-song-play-analysis
* Keypair: is353-song-play-analysis-keypair

#### Thêm IAM Roles cho Cluster
* Roles này cho phép truy cập và thay đổi dữ liệu trong bucket, quyền sử dụng các dịch vụ EMR, EC2.


### Chạy `etl.py` trong cụm
Sau Cluster sẵn sàng ta truy cập vô primary node
```
ssh -i ~/is353-song-play-analysis-keypair.pem hadoop@ec2-IP***.ap-southeast-1.compute.amazonaws.com
```

Mở terminal mới và đẩy 2 file [dl.cfg](dl.cfg) và [etl.py](etl.py) bằng lệnh:
```
scp -i ~/is353-song-play-analysis-keypair.pem etl.py hadoop@ec2-IP***.ap-southeast-1.compute.amazonaws.com:/home/hadoop
scp -i ~/is353-song-play-analysis-keypair.pem dl.cfg hadoop@ec2-IP***.ap-southeast-1.compute.amazonaws.com:/home/hadoop
```

Chạy mã spark:
```
spark-submit etl.py
```

### Kết quả
Ta thu được star schema lưu trữ dữ liệu giúp truy vấn dữ liệu nhanh hơn trong folder output trên bucket (đã clone vào [output](output))
