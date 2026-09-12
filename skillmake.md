# skillmake.md — Skill hướng dẫn patch/make UI trong lib `.so`

## Mục đích
File này dùng để hướng dẫn AI agent/dev cách phân tích và patch UI trong thư viện native `.so`, đặc biệt UI kiểu ImGui/ARM64. Trọng tâm là **make lại giao diện login/menu**, sửa lỗi form, đổi bố cục, thêm nút tương tác như `DÁN KEY`, nhưng vẫn giữ nguyên logic quan trọng: auth, key, HWID, callback, handler button, state menu và các chức năng đang có.

---

## Nguyên tắc chính

1. **Luôn backup file gốc**
   - Tạo bản copy trước khi patch.
   - Ghi lại SHA256 của file gốc và file sau patch.
   - Output nên đặt tên phiên bản riêng, ví dụ:
     - `libname_ui_v1.so`
     - `libname_redesign_v2.so`
     - `libname_top_tabs_v3.so`

2. **Patch UI, giữ logic**
   - Chỉ thay đổi phần vẽ giao diện, layout, text, màu, kích thước, tab/menu.
   - Giữ nguyên luồng kiểm tra key/HWID/API nếu người dùng chỉ yêu cầu UI.
   - Giữ nguyên handler của button: login, paste, unlock, tab select, checkbox, slider.

3. **Không đoán offset**
   - Offset phải lấy từ file đang patch.
   - Không dùng offset từ ảnh, tên file, hoặc bản lib khác nếu chưa xác minh.
   - Mỗi bản `.so` có thể lệch offset dù UI nhìn giống nhau.

4. **Ảnh chỉ là bằng chứng lỗi giao diện**
   - Screenshot giúp thấy lỗi: chồng chữ, sai font, nút lệch, form nhỏ, màu xấu.
   - Screenshot không tự chứng minh vị trí patch trong binary.

---

## Quy trình patch lib UI

### Bước 1: Kiểm tra file đầu vào

- Xác định kiến trúc:
  - ARM64 / AArch64
  - ARM32
  - x86/x64
- Xác định loại file:
  - ELF `.so`
  - stripped hay còn symbol
- Tính hash:

```bash
sha256sum libinput.so
```

Trên PowerShell:

```powershell
Get-FileHash .\libinput.so -Algorithm SHA256
```

---

### Bước 2: Tìm chuỗi UI

Tìm các text đang hiện trên giao diện:

```bash
strings -a libinput.so | grep -i "access"
strings -a libinput.so | grep -i "telegram"
strings -a libinput.so | grep -i "device"
strings -a libinput.so | grep -i "esp"
```

Các chuỗi thường dùng để định vị UI:

- `ACCESS KEY`
- `DAN KEY`
- `MỞ KHÓA`
- `DEVICE`
- `DEVICE CODE`
- `ACCESS GRANTED`
- `ESP`
- `AIM`
- `ITEM`
- `Function`
- `Telegram`

Khi đổi chuỗi trong `.rodata`, cần chú ý:

- Chuỗi mới phải có byte `00` kết thúc.
- Nếu chuỗi mới dài hơn slot cũ, cần tìm code cave/string cave mới.
- Không ghi đè sang chuỗi khác kế bên.

---

### Bước 3: Map hàm render UI

Dựa vào xref của chuỗi UI để tìm hàm render.

Ví dụ pattern thường gặp trong ImGui:

- `Begin(...)`
- `BeginChild(...)`
- `EndChild()`
- `Button(...)`
- `InputText(...)`
- `Text(...)`
- `SameLine(...)`
- `SetCursorPos(...)`
- `PushStyleColor(...)`
- `PopStyleColor(...)`
- `PushStyleVar(...)`
- `PopStyleVar(...)`

Mục tiêu map:

- Vùng login form.
- Vùng status/error/loading.
- Vùng success/access granted.
- Vùng main menu/tabs.
- Vùng handler khi bấm button.

---

## Cách make lại UI login

### Layout đề xuất

Dùng token chung để dễ sửa:

```text
WINDOW_W      = 720
WINDOW_H      = 520
PADDING       = 22
HEADER_H      = 86
INPUT_H       = 56
BUTTON_H      = 54
GAP           = 12
RADIUS        = 12
```

Cấu trúc đẹp, dễ nhìn:

```text
┌────────────────────────────────────┐
│              LOGO / ICON            │
│        TÊN LOGIN / AUTH TITLE       │
├────────────────────────────────────┤
│  ACCESS KEY INPUT                   │
│  [ DÁN KEY ]       [ MỞ KHÓA ]      │
│  status/error/loading area          │
├────────────────────────────────────┤
│  DEVICE        Redmi...             │
│  DEVICE CODE   xxxx...yyyy          │
├────────────────────────────────────┤
│  Telegram: @Username                │
└────────────────────────────────────┘
```

### Lỗi thường gặp cần sửa

- Text dính vào border input.
- Font quá nhỏ hoặc bị thiếu glyph.
- Nút nằm chồng lên input.
- Loading che error text.
- Footer/Telegram bị cắt.
- Dùng `SameLine` sai làm button dính nhau.
- Quên `PopStyleColor/PopStyleVar` làm hỏng màu toàn menu.

---

## Thêm nút `DÁN KEY`

Khi thêm button paste:

1. Button phải gọi clipboard handler thật.
2. Clipboard rỗng thì giữ nguyên key hiện tại.
3. Clipboard quá dài thì cắt đúng buffer.
4. Luôn thêm `NUL` cuối chuỗi.
5. Không tự gọi login sau khi paste.

Pseudo logic:

```c
if (Button("DAN KEY", pasteButtonSize)) {
    const char* clip = GetClipboardText();
    if (clip && clip[0]) {
        CopyToKeyBuffer(keyBuffer, clip, KEY_BUFFER_SIZE);
        keyBuffer[KEY_BUFFER_SIZE - 1] = '\0';
    }
}
```

---

## Make lại menu chính

### Đổi tab bên trái thành tab ngang trên đầu

Layout:

```text
┌──────────────────────────────────────────┐
│        >> LDK << Mod cheat               │
├──────────────────────────────────────────┤
│ [ ESP ] [ AIM ] [ Function ] [ ITEM ]    │
├──────────────────────────────────────────┤
│                                          │
│        Nội dung tab đang chọn             │
│        Giữ nguyên checkbox/slider         │
│        Giữ nguyên chức năng               │
│                                          │
└──────────────────────────────────────────┘
```

Nguyên tắc:

- Chỉ đổi bố cục tab.
- Handler tab chọn vẫn giữ state cũ.
- Nội dung từng tab giữ nguyên.
- Checkbox/slider/function call giữ nguyên.
- Nếu dùng ImGui, sau mỗi button tab cần đặt `SameLine` đúng vị trí.

Pseudo layout:

```c
Begin(" >> LDK << Mod cheat###ESP ");

BeginChild("top_tabs", ImVec2(0, topBarHeight), true);
float gap = 10.0f * scale;
float w = (GetContentRegionAvail().x - gap * 3.0f) / 4.0f;
float h = 60.0f * scale;

if (Button("ESP", ImVec2(w, h))) currentTab = TAB_ESP;
SameLine(0, gap);
if (Button("AIM", ImVec2(w, h))) currentTab = TAB_AIM;
SameLine(0, gap);
if (Button("Function", ImVec2(w, h))) currentTab = TAB_FUNCTION;
SameLine(0, gap);
if (Button("ITEM", ImVec2(w, h))) currentTab = TAB_ITEM;
EndChild();

BeginChild("content", ImVec2(0, 0), true);
RenderCurrentTabContent(currentTab);
EndChild();

End();
```

---

## Patch ARM64 in-place — checklist

Khi patch binary ARM64:

- Mọi lệnh branch phải đúng range.
- Giữ stack alignment 16 bytes.
- Giữ register quan trọng trước/sau call.
- Không phá x19-x29 nếu hàm gốc đang dùng.
- Float args ImGui thường đi qua `s0/s1` hoặc struct `ImVec2` tùy ABI/hàm wrapper.
- Nếu gọi helper ngoài, kiểm tra register bị clobber.
- Nếu dùng code cave, kiểm tra vùng đó thật sự trống hoặc đã được chuyển an toàn.

Protected ranges nên có:

```json
{
  "protected_ranges": [
    { "name": "auth_logic", "start": "OFFSET", "end": "OFFSET" },
    { "name": "key_hwid_check", "start": "OFFSET", "end": "OFFSET" },
    { "name": "button_handlers", "start": "OFFSET", "end": "OFFSET" },
    { "name": "feature_content", "start": "OFFSET", "end": "OFFSET" }
  ]
}
```

---

## Manifest patch nên có

Mỗi bản patch nên lưu manifest:

```json
{
  "input": "libinput.so",
  "output": "liboutput.so",
  "source_sha256": "SOURCE_HASH",
  "output_sha256": "OUTPUT_HASH",
  "patches": [
    {
      "name": "replace_title",
      "offset": "OFFSET",
      "old_hex": "OLD_BYTES",
      "new_hex": "NEW_BYTES",
      "reason": "Change visible window title"
    }
  ],
  "protected_ranges": [
    {
      "name": "auth_key_hwid_logic",
      "start": "OFFSET",
      "end": "OFFSET",
      "status": "unchanged"
    }
  ]
}
```

---

## Test sau khi patch

### 1. Static test

- So sánh diff byte.
- Kiểm tra patch đúng offset.
- Kiểm tra chuỗi có `NUL`.
- Kiểm tra protected ranges còn nguyên.
- Kiểm tra cân bằng style stack:
  - `PushStyleColor` / `PopStyleColor`
  - `PushStyleVar` / `PopStyleVar`
  - `BeginChild` / `EndChild`

### 2. Emulation test

Test các case:

- Không bấm gì.
- Bấm paste key.
- Bấm unlock/login.
- Bấm từng tab.
- Scale màn hình nhỏ/vừa/lớn.
- Clipboard rỗng/dài/bình thường.

### 3. Runtime test

Chạy thật trên app/game:

- Mở form login.
- Nhập key thủ công.
- Bấm `DÁN KEY`.
- Bấm `MỞ KHÓA`.
- Sai key hiển thị lỗi đúng vùng.
- Đúng key vào success/menu.
- Tab ESP/AIM/Function/ITEM đổi đúng.
- Chức năng cũ vẫn còn.

---

## Template trả kết quả cho người dùng

```text
Đã patch xong bản UI mới.

File output: PATH_OUTPUT
SHA256: OUTPUT_HASH

Đã đổi:
- Make lại login/menu UI.
- Thêm/sửa button theo yêu cầu.
- Đổi title/text/màu/layout.

Đã giữ:
- Auth key/HWID.
- Handler login/paste/tab.
- Các chức năng cũ trong menu.

Đã check:
- Static diff/hash.
- Protected ranges.
- Emulation/layout case.

Runtime trên máy thật: STATUS
```

---

## Ghi chú quan trọng

- Make lại UI nghĩa là đổi cấu trúc giao diện, không chỉ tăng font hoặc đổi màu.
- Với binary patch, luôn tạo bản mới và giữ bản rollback.
- Nếu lỗi UI do padding/scale, hãy tính lại geometry theo scale thật thay vì cộng offset tay.
- Nếu title ImGui đổi nhưng muốn giữ ID ổn định, dùng dạng:

```c
"Tên hiển thị mới###ID_CU"
```

Ví dụ:

```c
">> LDK << Mod cheat###ESP"
```

Phần trước `###` là text hiển thị, phần sau là ID nội bộ của ImGui.
