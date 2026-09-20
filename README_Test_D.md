# Technical Artist Test - [Test D - Shader Tech]

**Candidate:** Lương Duy Tuấn Phong
**Test:** D - Shader Tech
**Completion Time:** 5.0 Man-Days
**UE Version:** 5.8

#🎯 Test Overview

- Tôi đã xây dựng một hệ thống làm ướt dạng mô-đun trong Unreal Engine, sử dụng hai Material Function có thể tái sử dụng: MF_WetnessPuddles, MF_WetnessSurface và MF_SnowAccumulation, với các thông số thời tiết được điều khiển chung qua MPC_GlobalWeather_Wetness, BP_WeatherSystem là Blueprint điều khiển thời tiết realtime.

- MF_WetnessPuddles tạo puddle dựa trên height, slope mask và texture breakup tùy chọn, kèm các param điều chỉnh coverage, độ sắc nét ranh giới, falloff độ ướt, diện mạo nước, rain ripple động và wind-driven normal detail. Thách thức chính: kết hợp height/slope mask để kiểm soát vùng đọng nước, tạo transition mượt giữa puddle và bề mặt ướt xung quanh, và cân bằng normal blending với displacement.

- MF_WetnessSurface tạo hiệu ứng ướt bằng cách darken albedo, điều chỉnh roughness và thêm droplet cùng flowing streak chuyển động. Hàm hỗ trợ World-Aligned và Triplanar projection, với param điều chỉnh droplet intensity, texture tiling và animation speed. Thách thức chính: giữ projection nhất quán trên các bề mặt hướng khác nhau, hiệu chỉnh normal sau projection, blend chi tiết nước với normal gốc của material, và dùng time-varying mask để tạo hiệu ứng droplet fade in/out.

- MF_SnowAccumulation thêm lớp snow cho material hiện có, điều khiển chung qua MPC_GlobalWeather_Snow. Hàm kết hợp directional accumulation mask với texture breakup, cho phép chỉnh snow amount, độ sắc nét transition, màu sắc, roughness, normal intensity và displacement. Wind direction ảnh hưởng vùng tích tuyết qua MF_Snow_WindDirection, kèm Fresnel tùy chọn tạo sparkle nhẹ. Thách thức chính: tạo coverage tự nhiên trên các bề mặt hướng khác nhau, làm ranh giới tuyết bớt đều, và blend normal của tuyết với surface detail gốc. Pixel mask và vertex mask được tách riêng để phục vụ shading và displacement, cho phép điều chỉnh độc lập coverage và độ dày tuyết.

- NS_Snow là hệ thống Niagara kết hợp falling snowflake và drifting snow, dùng wind, turbulence và updraft để tạo chuyển động đa dạng. Hệ thống cho phép chỉnh spawn density, particle size, lifetime, collision và LOD theo khoảng cách. Thách thức chính: kết hợp nhiều lớp motion để tạo snow gust tự nhiên, xử lý vùng có che chắn (occlusion), và giữ hình ảnh nhất quán giữa particle gần và xa.

- NS_Rain là Niagara system kết hợp falling raindrop, rain sheet và surface splash, với param chỉnh density, speed, wind direction, particle look và splash intensity vẫn cho phép BP_WeatherSystem điều khiển cường độ mưa.

BP_WeatherSystem là Blueprint điều khiển trung tâm, phối hợp hiệu ứng mưa và tuyết với phản ứng của material trong môi trường. Blueprint điều chỉnh cường độ hạt Niagara và cập nhật MPC_GlobalWeather_Wetness cùng MPC_GlobalWeather_Snow thông qua các chuyển tiếp sử dụng Timeline. Các thách thức kỹ thuật chính gồm đồng bộ mưa/tuyết với độ ướt bề mặt và lượng tuyết tích tụ, hỗ trợ thời tiết xuất hiện rồi trở về trạng thái ban đầu một cách từ từ, đồng thời phối hợp chuyển tiếp của hạt và material với các thiết lập thời gian riêng.


# 🚀 Key Features

- Feature 1 — Unified Weather Control
  Implementation: Tôi xây dựng BP_WeatherSystem để điều phối NS_Rain và NS_Snow thông qua timeline-driven transitions, giúp việc chuyển đổi giữa các trạng thái thời tiết diễn ra mượt mà hơn thay vì bật/tắt đột ngột. Niagara parameters được dùng để kiểm soát precipitation intensity, trong khi MPC_GlobalWeather_Wetness và MPC_GlobalWeather_Snow giúp lan truyền hiệu ứng material ra toàn bộ scene một cách đồng bộ. Tôi tách riêng particle timeline và material timeline để rainfall, surface wetness và snow accumulation có thể phát triển với tốc độ khác nhau, học được cách tổ chức logic này giúp hệ thống linh hoạt hơn khi cần chỉnh sửa sau này.

- Feature 2 — Layered Precipitation Effects
  Performance considerations: Trong quá trình làm NS_Rain, tôi kết hợp raindrop, rain curtain và splash thành nhiều layer riêng biệt; tương tự với NS_Snow là falling flake, drifting snow và wind turbulence. Tôi áp dụng camera-relative spawning và distance-based snow detail để tập trung chi tiết hình ảnh quanh camera, từ đó tối ưu hiệu năng khi số lượng particle tăng lên. Đây cũng là phần tôi dành nhiều thời gian thử nghiệm nhất — điều chỉnh spawn rate, particle lifetime, collision settings và xử lý translucent particle overlap để cân bằng giữa chất lượng hình ảnh và chi phí rendering.

- Feature 3 — Reusable Material Integration
  Unreal Engine workflow: Tôi xây dựng MF_WetnessSurface, MF_WetnessPuddles và MF_SnowAccumulation dựa trên Material Attributes để có thể tái sử dụng và tích hợp vào các master material sẵn có mà không cần viết lại từ đầu. Material Instances giúp expose các control cần thiết để chỉnh riêng cho từng surface, còn Material Parameter Collections cho phép chia sẻ giá trị thời tiết chung giữa các material. Tôi thực hành quy trình điều chỉnh thông số ngay trong Material Instance Editor, sau đó preview kết quả phối hợp qua BP_WeatherSystem trong Play In Editor để kiểm tra hiệu ứng có hoạt động đúng như mong đợi hay không.

# 🎮 How to Test

1. Extract the project ZIP file.
2. Open the project using **Unreal Engine 5.8**.
3. Open and play the **TA_Test_B_D** level.
4. Press M to make it rain.
Press N to switch to snowy weather.
Press O to return to normal weather.
5. kiểm tra material của các model trong level có mưa/tuyết phủ .

# 📊 Performance Metrics 

- Target FPS: 60 | Achieved: 60
- Memory Usage:  6.93 GB RAM / 10.91 GB VRAM

- Frame Time: 16.67 ms
- GPU Time: >= 7.00 ms
 

# 🛠 Tools & Techniques Used
- UE5 Systems: Các hệ thống UE5: Niagara tạo hạt mưa và tuyết; Blueprint và Timeline điều khiển chuyển tiếp thời tiết; Material Function tạo độ ướt, vũng nước và tuyết tích tụ; Material Parameter Collection điều khiển thời tiết toàn cục; Material Instance tinh chỉnh riêng từng bề mặt.


## 🎨 Media Preview

![Screenshot 1](Media/Screenshots_Test_B/01_Overview.png)

![Screenshot 2](Media/Screenshots_Test_B/Technical_Detail1.png)
![Screenshot 3](Media/Screenshots_Test_B/Technical_Detail2.png)
![Screenshot 4](Media/Screenshots_Test_B/Technical_Detail3.png)
![Screenshot 5](Media/Screenshots_Test_B/Technical_Detail4.png)
![Screenshot 6](Media/Screenshots_Test_B/Technical_Detail5.png)
![Screenshot 7](Media/Screenshots_Test_B/03_UE_Editor_Screenshot.png)
![Screenshot 8](Media/Screenshots_Test_B/04_Performance_Metrics.png)

![Video Demo](Media/Videos_Test_B/Gameplay_Demo.mp4)
![Technical Showcase](Media/Videos_Test_B/Technical_Showcase.mp4)