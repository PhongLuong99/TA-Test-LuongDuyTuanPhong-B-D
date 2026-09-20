# Technical Breakdown — Weather System

**Phạm vi:** `MF_WetnessPuddles`, `MF_WetnessSurface`, `MF_SnowAccumulation`, `MPC_GlobalWeather_Wetness`, `BP_WeatherSystem`, `NS_Snow`, `NS_Rain`.

**Ghi chú:** Tài liệu này dựa trên việc tôi đọc tên node, tham số và comment trong các `.uasset` của project UE 5.8, cùng config và log PIE hiện có. Tôi chưa export đầy đủ graph nên phần vai trò asset là điều tôi quan sát trực tiếp, còn phần diễn giải luồng xử lý là suy luận của tôi dựa trên cấu trúc đó — không phải khẳng định 100% đúng với logic gốc. Các bước kiểm tra bên dưới là quy trình tôi đề xuất để tự xác nhận lại, chưa phải kết quả đã test.

## 1. Cách tổ chức hệ thống

Tôi chia hệ thống thành 3 phần: Blueprint điều phối trạng thái thời tiết, Material Function xử lý phản ứng bề mặt, và Niagara lo phần mưa/tuyết trong không gian. Cách chia này giúp tôi chỉnh mưa rơi độc lập với độ ướt còn đọng lại, hoặc chỉnh tuyết rơi độc lập với lớp tuyết phủ.

`BP_WeatherSystem` tham chiếu cả 2 Niagara System và 2 Material Parameter Collection (MPC). Nhánh wetness dùng `MPC_GlobalWeather_Wetness`, nhánh snow dùng thêm `MPC_GlobalWeather_Snow`.

## 2. MF_WetnessSurface — độ ướt, giọt nước, vệt chảy

Function này xử lý lớp nước mỏng trên bề mặt, tách thành 3 phần: đổi màu/roughness do ướt, thêm normal giọt nước, và tạo chuyển động vệt nước chảy. Tôi tách riêng để bề mặt vẫn trông ướt kể cả khi giảm hiệu ứng giọt nước.

Asset dùng texture `T_waterDroplets_01_N`, `T_waterDrips_01_N`, `T_rainDripsPacked_01_Mask`, `T_waterDroplets_Temporal_01_Mask`, cùng các node Time/Frac/Sine cho animation theo thời gian thay vì normal tĩnh.

Tôi hỗ trợ cả `World-Aligned Mapping` và `Triplanar Mapping` để dùng được trên nhiều loại mesh mà không phụ thuộc UV riêng, kèm `BlendAngleCorrectedNormals` để blend normal nền với chi tiết nước. tôi đã tham khảo Ai và tài liệu trên Internet

**Khó nhất:** khi bề mặt chuyển từ ngang sang đứng, giọt nước phải đổi cách xuất hiện, và khi dùng triplanar thì normal cũng phải đổi trục theo hướng chiếu — nếu blend thẳng, ánh sáng sẽ bị lệch. Tôi dùng VertexNormalWS + DotProduct để phân loại hướng bề mặt cho việc này.

**Cách tôi kiểm tra:** preview riêng mask/normal/kết quả cuối trong Material Function Editor, thử trên plane ngang, đứng và mesh xoay; sau đó chỉnh mapping, tiling, droplets amount trong Material Instance Editor để xem có tái sử dụng tốt không.

## 3. MF_WetnessPuddles — vũng nước đọng

Function này tách riêng vũng nước đọng khỏi độ ướt thông thường, dùng `MF_PuddleMask`, `MF_Rain_Ripples`, `T_GlobalPuddleMask_01`, `T_PuddleWind_01_N`.

Các nhóm tham số chính:

- Phân bố: `Puddle Coverage`, `Puddle Mask Scale`, `Puddle Sharpness`, `Use Puddle Breakup Texture`
- Điều kiện bề mặt: `Slope Angle Cutoff/Mask`, `Height Input/Mask/Mixing`
- Chuyển tiếp: `Puddle Wetness Falloff`, `Transition Mask`
- Đặc tính nước: `Puddle Color/Opacity`, `Water Specular`, `Puddle Normals`
- Chuyển động: rain ripples và wind ripples (amount/intensity/speed/tiling riêng)

Tôi nghĩ lý do phải kết hợp height + slope + breakup texture là vì nếu chỉ dùng 1 mask đồng nhất thì puddle sẽ không tự nhiên trên bề mặt có địa hình.

**Khó nhất:** puddle lỡ xuất hiện trên mặt dốc, mép puddle bị cứng, và mặt nước giữ nguyên chi tiết normal lởm chởm của nền. Tôi xử lý bằng slope mask, wetness falloff và FlattenNormal.

**Cách tôi kiểm tra:** preview height mask → slope mask → mask cuối, thử trên bề mặt bằng/nghiêng/height khác nhau, rồi tăng coverage dần để xem vùng nước mở rộng đúng không.

## 4. MF_SnowAccumulation — lớp tuyết phủ

Function dùng `MPC_GlobalWeather_Snow`, `MF_Snow_WindDirection`, `T_snowMask_01_M`, `T_snowTile_01_N`, với tham số `Snow Amount`, `Snow Fade Control`, `Transition Sharpness`, `Breakup Intensity/Tiling`, `Snow Color/Roughness/Normal Intensity`, `Snow Displacement Multiplier`.

Tôi làm cả `Per Vertex Snow Mask` và `Per Pixel Snow Mask` vertex mask lo hướng tổng thể của hình học, pixel mask lo chi tiết normal sát bề mặt. Ý tưởng chính là tính độ thuận hướng giữa normal bề mặt và hướng gió tích tụ, kiểu `saturate(dot(N, D))`, rồi remap qua amount/fade/sharpness/breakup.

Lý do tôi xử lý snow ở material thay vì mô phỏng từng hạt rơi rồi cộng dồn: material tái sử dụng được trên nhiều đối tượng, còn Niagara chỉ lo phần hạt tuyết đang rơi trong không khí 2 phần này chạy độc lập, tốc độ khác nhau cũng được.

**Khó nhất:** chỉ dùng normal tổng thể thì mép tuyết quá đều, chỉ dùng normal chi tiết thì bị nhiễu — nên tôi kết hợp cả 2 loại mask + breakup texture + transition sharpness để cân bằng. Vấn đề tôi chưa giải quyết được là kiểm tra mái che: hướng normal không tự động biết bề mặt nào ở trong nhà để không phủ tuyết lên đó.

**Cách tôi kiểm tra:** preview mask trên mặt ngang/đứng/vật xoay, chỉnh amount xem tiến trình phủ, và tách riêng nhánh Fresnel (`Snow Fresnel Toggle`, `Glimmer Mask`) khi đánh giá highlight đây chỉ là hiệu ứng lấp lánh nhẹ, không phải mô hình tán xạ tuyết vật lý đầy đủ.

## 5. MPC_GlobalWeather_Wetness — dữ liệu dùng chung

Collection có 3 nhóm tham số: độ ướt (`Surface Wetness Amount`, `Wet Surface Darkening`, `Global Effect Droplets`), puddle (`Puddle Coverage/Mask Scale/Opacity/Sharpness/Wetness Falloff/Color`), và ripple (rain/wind ripple amount/intensity/speed/tiling).

MPC chỉ là nguồn giá trị chung phần tạo mask và shading thực sự nằm trong các Material Function. Blueprint update giá trị này qua node `SetScalarParameterValue`.

**Giới hạn tôi nhận ra:** có tham số không có nghĩa là Blueprint đang animate hết tất cả cần xem graph đầy đủ mới chắc.

## 6. BP_WeatherSystem — Blueprint điều phối

Blueprint có biến `ActiveWeather` (enum `E_WeatherTarget`), component `Rain Particles`/`Snow Particle`, và 4 Timeline: `Rain_Timeline`, `Snow Timeline`, `Material_Timeline_R`, `Material Timeline Snow` — mỗi cái có 1 track riêng.

Tôi hiểu cấu trúc này như một controller dùng Timeline để biến đổi hiệu ứng theo thời gian: nhánh Niagara cập nhật mức hạt (`SetNiagaraVariableFloat`), nhánh material cập nhật MPC (`SetScalarParameterValue`), qua Lerp: `Value(t) = Lerp(Min, Max, Alpha(t))`.

Tôi tách riêng Timeline material và particle để có thể tạo độ trễ ví dụ mưa bắt đầu rơi trước, rồi puddle mới dần hình thành sau, thay vì bật cùng lúc. Blueprint còn có DirectionalLight, SkyLight, ExponentialHeightFog, VolumetricCloud và `MI_Cloud_Custom`, nên phạm vi rộng hơn chỉ bật/tắt particle dù tôi chưa xác nhận được mọi thuộc tính ánh sáng/mây/fog có thực sự được animate hết hay không.

**Khó nhất:** nếu chuyển từ mưa sang tuyết giữa chừng, nhiều Timeline có thể cùng ghi giá trị hoặc bị nhảy khi phát lại từ đầu. `ActiveWeather` và các nhánh Reverse giúp tổ chức việc này nhưng tôi chưa test kỹ cơ chế chống xung đột. Việc cần làm tiếp: test đổi trạng thái liên tục, đảo chiều giữa Timeline, và đếm số Niagara component sau nhiều lần chuyển để chắc không bị leak.

`Begin Rain` và `Begin Snow`, xác nhận 2 nhánh này từng chạy được nhưng chưa chứng minh chuyển tiếp mượt hay đạt hiệu năng mong muốn.

## 10. Tổ chức asset

```text
Content/
  Blueprint/
    BP_WeatherSystem
    Enums/E_WeatherTarget
  Material/
    Material_Funtion/
      MF_WetnessSurface
      MF_WetnessPuddles
      MF_SnowAccumulation
      Function/          # mask, ripple, hướng gió, normal swizzle
    MPC/
    Material_NS_Rain/
    Material_NS_Snow/
  Niagara/
    NS_Rain
    NS_Snow
    Moduel/
  Texture/
```


