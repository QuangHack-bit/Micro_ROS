# CHẤM DỨT VIỆC PHẢI LÀM LẠI MỖI LẦN RÚT/CẮM

## Vì sao bạn phải làm lại (nguyên nhân gốc)

Windows tạo **một device instance mới cho mỗi cổng USB vật lý**. Bạn cắm adapter lúc thì cổng này, lúc thì cổng khác → BUSID nhảy `2-11` ↔ `2-13` → binding cũ không áp dụng → `Not shared` → phải `bind` lại bằng Admin.

**Hai việc chấm dứt chuyện này:**
1. **Cố định MỘT cổng USB vật lý** cho adapter (dán nhãn/băng dính đánh dấu cổng đó)
2. Dùng `--hardware-id` + `--auto-attach` thay vì `--busid`

Sau khi làm đủ 2 việc này: **rút/cắm bao nhiêu lần cũng không cần gõ lại lệnh nào.**

---

# 📋 BẢNG LỆNH THEO 3 SHELL

| Shell | Khi nào cần | Lệnh |
|---|---|---|
| 🛡️ **[PS-Admin]** | **Chỉ 1 lần duy nhất** (hoặc khi `usbipd list` báo `Not shared`) | `usbipd bind --hardware-id 067b:2303` |
| 🪟 **[PS-Thường]** | **Mỗi lần khởi động máy / sau `wsl --shutdown`** — mở 1 cửa sổ riêng, **GIỮ MỞ** | `usbipd attach --wsl --hardware-id 067b:2303 --auto-attach` |
| 🐧 **[WSL]** | Mỗi lần muốn kiểm tra | `amr-check` (script tạo bên dưới) |
| 🐧 **[WSL]** | Khi rút/cắm lại adapter | **KHÔNG CẦN GÌ** — auto-attach tự lo |

> ⚠️ `bind` là **persistent, sống qua reboot**. Bạn **không cần Admin nữa** trừ khi đổi cổng USB vật lý.

---

# 🔧 THIẾT LẬP 1 LẦN — 3 KHỐI LỆNH

## Khối 1 🛡️ **[PS-Admin]** — chạy đúng 1 lần

Start → `powershell` → **Run as Administrator**:

```powershell
# Cam adapter vao PORT CO DINH ma ban se dung mai mai
usbipd bind --hardware-id 067b:2303
usbipd list
```

**Yêu cầu:** dòng `067b:2303` phải hiện `Shared`. Ghi nhớ BUSID lúc này (ví dụ `2-11`) và **từ nay chỉ cắm vào đúng cổng đó**.

Đóng cửa sổ Admin. Không cần mở lại nữa.

---

## Khối 2 🪟 **[PS-Thường]** — tạo shortcut để khỏi gõ

Start → `powershell` → Enter (**không** admin). Dán 1 lần:

```powershell
@'
Write-Host "=== AMR: PL2303 -> WSL (GIU CUA SO NAY MO) ===" -ForegroundColor Cyan
Write-Host "Rut/cam adapter thoải mai, loop nay tu attach lai." -ForegroundColor Yellow
usbipd attach --wsl --hardware-id 067b:2303 --auto-attach
'@ | Set-Content -Path "$env:USERPROFILE\amr-attach.ps1" -Encoding UTF8
```

**Từ nay mỗi lần mở máy:** mở File Explorer → `C:\Users\DELL\` → chuột phải `amr-attach.ps1` → **Run with PowerShell**. Giữ cửa sổ đó mở suốt buổi làm việc.

Nếu muốn chạy thẳng không cần file:
```powershell
usbipd attach --wsl --hardware-id 067b:2303 --auto-attach
```

**Tuỳ chọn bền hơn** (nếu bạn cố định cổng và muốn loop sống cả khi adapter đang rút):
```powershell
usbipd attach --wsl --busid 2-11 --auto-attach --unplugged
```
> Cờ `--unplugged` chỉ dùng được với `--busid` (theo `--help` bạn gửi). Đánh đổi: phải đúng BUSID cố định.

---

## Khối 3 🐧 **[WSL]** — tạo bộ lệnh dùng mãi mãi

Dán **toàn bộ** khối này 1 lần:

```bash
mkdir -p ~/bin

# ---------- 1. Udev: ten cong ON DINH /dev/amr_base ----------
sudo tee /etc/udev/rules.d/99-amr.rules >/dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{idVendor}=="067b", ATTRS{idProduct}=="2303", SYMLINK+="amr_base", MODE="0666", GROUP="dialout"
EOF
sudo udevadm control --reload 2>/dev/null || sudo service udev restart
sudo udevadm trigger 2>/dev/null

# ---------- 2. amr-check : trang thai toan bo duong truyen ----------
cat > ~/bin/amr-check <<'EOF'
#!/usr/bin/env bash
echo "--- USB ---";    lsusb | grep -i "067b:2303" || echo "  KHONG thay PL2303 -> xem cua so PS auto-attach"
echo "--- PORT ---";   ls -l /dev/amr_base /dev/ttyUSB* 2>/dev/null || echo "  CHUA co cong"
echo "--- DRIVER ---"; lsmod | grep -q pl2303 && echo "  pl2303 da nap" || echo "  pl2303 CHUA nap"
echo "--- DMESG ---";  sudo dmesg | grep -iE "pl2303|ttyUSB" | tail -4
echo "--- HOLDER ---"; sudo fuser -v /dev/ttyUSB0 2>/dev/null || echo "  cong trong"
echo "--- GROUP ---";  id -nG | tr ' ' '\n' | grep -qx dialout && echo "  dialout OK" || echo "  THIEU dialout"
EOF

# ---------- 3. amr-wait : cho cong san sang (toi da 60s) ----------
cat > ~/bin/amr-wait <<'EOF'
#!/usr/bin/env bash
for i in $(seq 1 60); do
  for p in /dev/amr_base /dev/ttyUSB0; do
    [ -e "$p" ] && { echo "SAN SANG: $p"; exit 0; }
  done
  sleep 1
done
echo "TIMEOUT: khong co cong. Kiem tra cua so PS auto-attach."; exit 1
EOF

# ---------- 4. amr-agent : chay agent, tu chon cong ----------
cat > ~/bin/amr-agent <<'EOF'
#!/usr/bin/env bash
P=/dev/ttyUSB0; [ -e /dev/amr_base ] && P=/dev/amr_base
[ -e "$P" ] || { echo "KHONG co cong. Chay amr-check."; exit 1; }
echo "Agent tren $P @115200 -v6"
exec ros2 run micro_ros_agent micro_ros_agent serial --dev "$P" -b 115200 -v6
EOF

# ---------- 5. amr-sniff : bat raw byte, co chot chan ----------
cat > ~/bin/amr-sniff <<'EOF'
#!/usr/bin/env bash
P=/dev/ttyUSB0; [ -e /dev/amr_base ] && P=/dev/amr_base
[ -e "$P" ] || { echo "KHONG co cong. Chay amr-check."; exit 1; }
sudo stty -F "$P" 115200 raw -echo -crtscts
PORT="$P" DUR="${1:-25}" python3 - <<'PY'
import serial, time, os
s = serial.Serial(os.environ['PORT'], 115200, timeout=0.05)
dur=float(os.environ['DUR']); t0=time.time(); last=None; nb=0; tot=0; nz=0
print(">>> BAM RESET STM32 NGAY <<<", flush=True)
while time.time()-t0 < dur:
    d = s.read(512)
    if d:
        t=time.time()-t0; tot+=len(d); nz+=sum(b!=0 for b in d); nb+=1
        gap=f"{(t-last)*1000:.0f}ms" if last else "-"
        last=t
        print(f"{t:6.2f}s gap={gap:>7} +{len(d):3d}B {d[:24].hex(' ')}", flush=True)
print(f"\nTONG {tot} byte | {nb} burst | byte khac 0x00: {nz}")
print("Framing 0x7E:", "CO" if nz else "KHONG -> loi tang bit")
PY
EOF

# ---------- 6. amr-loop : test loopback adapter ----------
cat > ~/bin/amr-loop <<'EOF'
#!/usr/bin/env bash
P=/dev/ttyUSB0; [ -e /dev/amr_base ] && P=/dev/amr_base
[ -e "$P" ] || { echo "KHONG co cong."; exit 1; }
echo ">>> RUT STM32, NOI TAT TX<->RX CUA ADAPTER <<<"
PORT="$P" python3 - <<'PY'
import serial, time, os
s=serial.Serial(os.environ['PORT'],115200,timeout=1); s.reset_input_buffer()
t=bytes([0x7E,0x00,0x55,0xFF,0x7D,0x01,0x7E])
s.write(t); s.flush(); time.sleep(0.3); g=s.read(len(t))
print("GUI :", t.hex(' ')); print("NHAN:", g.hex(' ') if g else "(rong)")
print("KET QUA:", "PASS" if g==t else "FAIL")
PY
EOF

chmod +x ~/bin/amr-*

# ---------- 7. Nap ~/bin vao PATH ----------
grep -q 'HOME/bin' ~/.bashrc || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/bin:$PATH"

echo "=== XONG. Cac lenh: amr-check | amr-wait | amr-agent | amr-sniff | amr-loop ==="
```

---

# 🔁 QUY TRÌNH HẰNG NGÀY — CHỈ 2 BƯỚC

| Bước | Shell | Việc |
|---|---|---|
| 1 | 🪟 **[PS-Thường]** | Chuột phải `amr-attach.ps1` → **Run with PowerShell**. **Giữ cửa sổ mở.** |
| 2 | 🐧 **[WSL]** | `amr-check` → thấy `/dev/amr_base` hoặc `/dev/ttyUSB0` là xong |

**Rút/cắm adapter lại:** không làm gì cả. Cửa sổ PS tự attach lại trong ~5 giây. Kiểm tra bằng `amr-check`.

---

# 🧰 CÁC LỆNH SẼ DÙNG ĐI DÙNG LẠI

🐧 **[WSL]**
```bash
amr-check          # trạng thái USB / cổng / driver / ai giữ cổng
amr-wait           # chờ cổng xuất hiện (dùng trong script)
amr-loop           # test loopback adapter (nhớ rút STM32, nối tắt TX-RX)
amr-sniff 25       # bắt raw byte 25s, tự nhắc bấm RESET
amr-agent          # chạy micro_ros_agent -v6 trên đúng cổng
```

---

# ⚠️ 4 LƯU Ý QUAN TRỌNG

1. **Cửa sổ PS chạy `--auto-attach` phải GIỮ MỞ.** Đó là vòng lặp vô hạn, đóng là mất tự động.
2. **`wsl --shutdown` làm mất mọi attachment** → chạy lại `amr-attach.ps1`.
3. **Chỉ cắm adapter vào ĐÚNG cổng đã đánh dấu.** Đổi cổng → phải 🛡️ **[PS-Admin]** `usbipd bind --hardware-id 067b:2303` lại.
4. **KHÔNG attach ST-Link (`0483:3748`) vào WSL.** CubeIDE trên Windows cần nó để debug. Lệnh `--hardware-id 067b:2303` chỉ khớp adapter nên an toàn.

---

# 🚑 XỬ LÝ SỰ CỐ NHANH

| Triệu chứng (`amr-check`) | Shell | Lệnh |
|---|---|---|
| Không thấy `067b:2303` trong `lsusb` | 🪟 PS-Thường | Xem cửa sổ auto-attach còn mở không. Nếu đóng → chạy lại |
| 🪟 `usbipd list` báo `Not shared` | 🛡️ PS-Admin | `usbipd bind --hardware-id 067b:2303` |
| Có `lsusb` nhưng không có `/dev/ttyUSB0` | 🐧 WSL | `sudo modprobe pl2303` rồi `amr-check` |
| `Device or resource busy` | 🐧 WSL | `sudo fuser -k /dev/ttyUSB0` |
| `Permission denied` | 🐧 WSL | `sudo usermod -aG dialout $USER` → 🪟 `wsl --shutdown` → mở lại |
| BUSID nhảy liên tục | — | **Cắm cố định 1 cổng vật lý** |

---

Sau khi dựng xong, chạy 🐧 `amr-check` gửi tôi output để xác nhận hạ tầng đã ổn định. Rồi ta quay lại việc còn dang dở: **`amr-loop` (test loopback) — bước bắt buộc chưa làm**, để phân định adapter clone có lỗi hay không trước khi động vào firmware.
