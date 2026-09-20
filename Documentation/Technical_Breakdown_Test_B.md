# Technical Breakdown - Character Animation Tech (TestB)

- Hệ thống được xây dựng trên nền Third Person Character có sẵn của Unreal Engine, tôi bổ sung khả năng trang bị, cất và chuyển đổi giữa Sword, Staff và Bow. BP_ThirdPersonCharacter lo phần gameplay, ABP_Manny lo animation, còn AnimNotify giúp đồng bộ vị trí vũ khí với động tác nhân vật. Nhân vật hiện dùng ABP_Quinn — một Animation Blueprint tôi cho kế thừa logic từ ABP_Manny thay vì viết lại từ đầu.

## 1. Cách tôi tổ chức hệ thống

Tôi chia làm 3 phần riêng biệt: BP_ThirdPersonCharacter nhận input và chọn Montage; ABP_Manny biến dữ liệu chuyển động + trạng thái vũ khí thành pose; ba Notify (AN_AttachSword/Staff/Bow) lo việc gắn vũ khí đúng lúc trong animation. Chia vậy để sau này tôi đổi thời điểm rút vũ khí mà không phải đụng vào input, hoặc đổi bộ locomotion mà không phải sửa lại phần gắn vũ khí.

Tôi dùng enum E_CombatType thay vì để nhiều biến boolean rời rạc — vừa rõ ràng vừa dễ dùng với Blend Poses by Enum để chọn animation. Việc chuyển vũ khí dùng Animation Montage, và AnimNotify đánh dấu thời điểm đổi socket (tay ↔ vị trí cất) để bám sát animation thay vì set 1 khoảng Delay cứng.

## 2. Cấu trúc Blueprint

BP_ThirdPersonCharacter dùng Enhanced Input cho di chuyển, camera, nhảy, chọn vũ khí; có CombatType, WeaponStatus, isWeaponSwitching để quản lý trạng thái. Luồng tôi hình dung khi đổi vũ khí:

Input chọn vũ khí → check trạng thái hiện tại → chọn Montage phù hợp → phát Montage → Notify đổi attachment → cập nhật trạng thái animation → xong.

(Đây là cách tôi hiểu kiến trúc, thứ tự update enum chính xác thì cần soi lại graph mới chắc.)

Character BP và AnimBP giao tiếp qua interface BPI_Animations (hàm UpdateCombatType) — làm vậy để không phải access thẳng biến nội bộ của AnimBP từ bên ngoài.

Trong ABP_Manny, tôi dùng Velocity/GroundSpeed/IsFalling cho locomotion, State Machine xử lý đứng/đi/chạy/nhảy/rơi/tiếp đất, và Blend Space riêng cho từng vũ khí (ABS_Sword/Staff/Bow) chọn qua Blend Poses by Enum. Layered Blend per Bone + slot UpperBody giúp phối trộn thao tác tay trên với locomotion mà không đụng chân.

Cả 3 Notify đều làm giống nhau: lấy MeshComp → lấy Owner → cast sang BP_ThirdPersonCharacter → lấy component vũ khí → gọi AttachToComponent. Biến AttachToHand quyết định gắn vào tay hay vị trí cất, nên 1 Notify dùng được cho cả equip lẫn unequip.

Điểm mình thấy chưa tối ưu: Notify đang cast cứng sang class nhân vật, nên nếu sau này có nhiều loại character thì phải tách logic attach ra interface/component riêng.

## 3. Workflow mình dùng trong UE

- Enhanced Input + Blueprint Editor: setup Input Action trong IMC_Default, xử lý request đổi vũ khí.
- Animation Blueprint Editor: dựng State Machine, nối Blend Space theo CombatType.
- Montage Editor: set slot UpperBody, chỉnh blend-in/out, đặt Notify đúng lúc tay chạm/thả vũ khí.
- Skeleton Editor: canh socket cho khớp tay và vị trí cất.
- Asset Override Editor: đổi animation cho ABP_Quinn mà vẫn dùng chung graph của ABP_Manny.

Để test, tôi dùng PIE + Debug Filter để theo dõi CombatType, isWeaponSwitching, state hiện tại và Montage đang chạy.

## 4. Những chỗ khó và cách tôi xử lý

- **Timing attachment:** gắn sớm/muộn là mesh tách khỏi tay ngay. Mình tách 2 lỗi riêng — sai vị trí thì sửa socket, sai thời điểm thì sửa Notify — rồi scrub animation để canh lại cho khớp.
- **Trang bị vũ khí mà không phá dáng đi:** dùng UpperBody + Layered Blend per Bone để giới hạn ảnh hưởng chỉ ở phần thân trên, test kỹ lúc đứng/đi/chạy để bắt lỗi lệch vai hay xoắn thân.
- **Đổi vũ khí liên tục:** isWeaponSwitching giúp chặn request mới khi đang switch, nhưng case Montage bị ngắt giữa chừng thì mình vẫn cần kiểm tra kỹ hơn để chắc nhân vật không bị kẹt trạng thái.
- **Đồng bộ gameplay và hình ảnh:** phải test đủ các case từ tay không ra trang bị, cất vũ khí, đổi thẳng giữa từng cặp Sword-Staff-Bow, đặc biệt lúc Montage đang blend.
- **Hiệu năng:** attach chạy theo Notify nên không phải check mỗi frame, đỡ tốn — nhưng chi phí State Machine/Blend Space/layered blend khi nhiều nhân vật cùng lúc thì mình chưa đo, cần profiling thêm.

## 5. Cách mình tổ chức asset

```text
Content/Blueprint/                      # Character Blueprint
Content/Blueprint/Enums/                # E_CombatType
Content/Blueprint/Interfaces/           # BPI_Animations
Content/Blueprint/AnimNotify/           # 3 Notify attach vũ khí
Content/Blueprint/Input/                # Mapping Context, Input Actions
Content/Characters/Mannequins/Animations/  # AnimBP, Blend Space, Montage
Content/Assets/Weapon/                  # mesh vũ khí
```

Quy trình thêm 1 vũ khí mới mình dự định: chuẩn bị mesh → canh socket → check animation khớp skeleton → tạo Blend Space/Montage → đặt Notify → nối trạng thái vào Character + AnimBP. Đổi animation thì phải test lại timing Notify vì thời điểm tay chạm vũ khí có thể lệch.



