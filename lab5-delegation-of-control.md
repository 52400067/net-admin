# LAB 5: ỦY QUYỀN QUẢN TRỊ (Delegation of Control)

Hướng dẫn làm từ đầu trên Windows Server mới.

## Tổng quan

**Ý nghĩa thực tế:** trong doanh nghiệp, Domain Admin không thể làm hết mọi việc như tạo tài khoản hay reset mật khẩu. Ta giao một phần quyền cho trưởng bộ phận, nhưng chỉ trong OU của họ. Vừa giảm tải cho IT, vừa an toàn (nguyên tắc đặc quyền tối thiểu). Bài này làm trọn vẹn từ máy mới: dựng DC, tạo OU, user, group, cài RSAT, ủy quyền và kiểm chứng.

| Máy | Hệ điều hành | IP ví dụ | DNS |
|---|---|---|---|
| FIT-DC-02 | Windows Server mới cài | 192.168.122.10 | 127.0.0.1 (chính nó) |
| FIT-WIN-01 | Windows 10/11 | 192.168.122.20 | 192.168.122.10 (trỏ về DC) |

> **Lưu ý:** Hai máy phải cùng một mạng ảo (ví dụ NAT) để thấy nhau và có Internet cho việc cài RSAT. Gateway mặc định của mạng NAT libvirt là `192.168.122.1`; đổi lại nếu mạng của bạn khác. Mọi lệnh chạy trong PowerShell mở bằng **Run as Administrator**.

**Các bước:** A. Dựng Domain Controller → B. Tạo OU, User, Group → C. Cho máy Client vào miền → D. Cài RSAT → E. Ủy quyền → F. Kiểm chứng.

---

## A. Dựng Domain Controller trên FIT-DC-02

### A1. Đặt IP tĩnh và đổi tên máy

```powershell
$nic = (Get-NetAdapter | Where-Object Status -eq "Up")[0].Name
New-NetIPAddress -InterfaceAlias $nic -IPAddress 192.168.122.10 `
 -PrefixLength 24 -DefaultGateway 192.168.122.1
Set-DnsClientServerAddress -InterfaceAlias $nic -ServerAddresses 127.0.0.1
net user Administrator P@ssw0rd
Rename-Computer -NewName "FIT-DC-02" -Restart
```

Máy khởi động lại. Đăng nhập lại bằng `Administrator` (mật khẩu `P@ssw0rd`). Nếu tên máy đã đúng, bỏ qua lệnh đổi tên.

### A2. Cài AD DS và nâng cấp thành Domain Controller

```powershell
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
```

```powershell
Install-ADDSForest -DomainName "labtdtu.com" -DomainNetbiosName "LABTDTU" `
 -InstallDns -SafeModeAdministratorPassword `
 (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) -Force
```

Chờ vài phút, máy tự khởi động lại. Cảnh báo về DNS delegation là bình thường.

### A3. Kiểm tra sau khi khởi động lại

Đăng nhập `LABTDTU\Administrator` (`P@ssw0rd`), chờ khoảng 2 phút rồi chạy:

```powershell
Get-Service ADWS, NTDS, DNS, Netlogon | Select Name, Status
Import-Module ActiveDirectory
Get-ADDomain | Select DNSRoot, NetBIOSName
```

**Đạt:** 4 dịch vụ ở trạng thái Running và hiển thị `labtdtu.com`.

### A4. Cho DC truy cập Internet (để Client cài được RSAT)

```powershell
Add-DnsServerForwarder -IPAddress 8.8.8.8
```

---

## B. Tạo OU, User, Group (trên FIT-DC-02)

Chạy một khối lệnh duy nhất. Khối này tạo OU, 4 user, 5 group và thêm `phungcp` vào `Officer`.

```powershell
Import-Module ActiveDirectory
$root = "DC=labtdtu,DC=com"
$k = "OU=Khoa_CNTT,$root"
$pwd = ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force

# 1. OU
New-ADOrganizationalUnit -Name "Khoa_CNTT" -Path $root
foreach ($ou in "Qualification","Office","Leader") {
 New-ADOrganizationalUnit -Name $ou -Path $k
}

# 2. User
$users = @(
 @{N="Truong Dinh Tu"; S="tutd"; O="Qualification"},
 @{N="Le Anh Cuong"; S="cuongla"; O="Qualification"},
 @{N="Tran Thanh Phuoc"; S="phuoctt"; O="Qualification"},
 @{N="Cao Phi Phung"; S="phungcp"; O="Office"}
)
foreach ($u in $users) {
 New-ADUser -Name $u.N -SamAccountName $u.S `
 -UserPrincipalName "$($u.S)@labtdtu.com" -Path "OU=$($u.O),$k" `
 -AccountPassword $pwd -Enabled $true -PasswordNeverExpires $true
}

# 3. Group
foreach ($g in "MangMayTinh","KHMayTinh","CNPhanMem") {
 New-ADGroup -Name $g -GroupScope Global -GroupCategory Security `
 -Path "OU=Qualification,$k"
}
New-ADGroup -Name "Officer" -GroupScope Global -GroupCategory Security -Path "OU=Office,$k"
New-ADGroup -Name "Leadership" -GroupScope Global -GroupCategory Security -Path "OU=Leader,$k"
Add-ADGroupMember -Identity "Officer" -Members "phungcp"
```

### Kiểm tra

```powershell
Get-ADOrganizationalUnit -Filter * | Select Name
Get-ADUser -Filter * -SearchBase $k | Select SamAccountName, DistinguishedName
Get-ADGroup -Filter * -SearchBase $k | Select Name
Get-ADGroupMember Officer | Select SamAccountName
```

**Đạt:** thấy 4 OU, 4 user đúng OU, 5 group, và `phungcp` trong `Officer`.

---

## C. Cho máy Client FIT-WIN-01 vào miền

Trên FIT-WIN-01, đăng nhập tài khoản local admin, mở PowerShell **Run as Administrator**:

```powershell
$nic = (Get-NetAdapter | Where-Object Status -eq "Up")[0].Name
New-NetIPAddress -InterfaceAlias $nic -IPAddress 192.168.122.20 `
 -PrefixLength 24 -DefaultGateway 192.168.122.1
Set-DnsClientServerAddress -InterfaceAlias $nic -ServerAddresses 192.168.122.10
Test-Connection 192.168.122.10 -Count 2
Add-Computer -DomainName labtdtu.com -Credential LABTDTU\Administrator -Restart
```

Khi hỏi mật khẩu, nhập `P@ssw0rd`. Máy khởi động lại. Đăng nhập bằng `labtdtu\Administrator`.

> **Lưu ý:** Nếu Client đã có IP thì bỏ lệnh `New-NetIPAddress`, chỉ cần đặt DNS trỏ về DC. Nếu join miền lỗi, kiểm tra lại DNS và lệnh ping tới DC.

---

## D. Cài RSAT trên FIT-WIN-01

Đăng nhập `labtdtu\Administrator`, mở PowerShell **Run as Administrator**:

```powershell
Add-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
Get-WindowsCapability -Online -Name "Rsat.ActiveDirectory*" | Select Name, State
```

**Đạt:** `State` là `Installed`. Nhấn `Win + R`, gõ `dsa.msc` để mở ADUC. Công cụ này cũng xuất hiện trong menu Start.

> **Lưu ý:** Nếu lỗi `0x800f0954` do máy dùng WSUS, chạy lệnh dưới đây rồi cài lại:

```powershell
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name UseWUServer -Value 0
Restart-Service wuauserv
```

---

## E. Ủy quyền Delegation of Control (giao diện ADUC)

Làm trên FIT-WIN-01 bằng tài khoản Administrator, trong `dsa.msc`. Mở rộng `labtdtu.com` rồi `Khoa_CNTT`.

### E1. OU Qualification

1. Chuột phải **Qualification**, chọn **Delegate Control...**, nhấn **Next**.
2. Nhấn **Add**, nhập `tutd; cuongla; phuoctt`, nhấn **Check Names**, **OK**, **Next**.
3. Chọn **Delegate the following common tasks**, tích **Create, delete, and manage user accounts**, **Next**, **Finish**.

### E2. OU Office

1. Chuột phải **Office**, chọn **Delegate Control...**, **Next**.
2. **Add**, nhập `phungcp`, **Check Names**, **OK**, **Next**.
3. Tích **Create, delete, and manage user accounts**, **Next**, **Finish**.

---

## F. Kiểm chứng

### F1. Với tài khoản phungcp

1. Đăng xuất Administrator, đăng nhập `labtdtu\phungcp` / `P@ssw0rd`.
2. Mở `dsa.msc`.
3. Trong OU **Office**: New → User, tạo thử `test01` (mật khẩu ví dụ `P@ssw0rd123`). Phải **thành công**.
4. Trong OU **Qualification** hoặc **Leader**: New → User. Phải bị **Access Denied**.

### F2. Với tài khoản tutd

1. Đăng xuất, đăng nhập `labtdtu\tutd` / `P@ssw0rd`, mở `dsa.msc`.
2. Tạo user trong OU **Qualification**: thành công.
3. Tạo user trong OU **Office** hoặc **Leader**: Access Denied.

| Tài khoản | Qualification | Office | Leader |
|---|---|---|---|
| tutd, cuongla, phuoctt | Thành công | Từ chối | Từ chối |
| phungcp | Từ chối | Thành công | Từ chối |

> **Lưu ý:** Chụp màn hình khi tạo thành công và khi bị Access Denied, nhớ để thấy rõ tên tài khoản đang đăng nhập.

---

## Lỗi thường gặp

| Hiện tượng | Cách xử lý |
|---|---|
| Không tìm thấy module ActiveDirectory | `Install-WindowsFeature RSAT-AD-PowerShell` trên DC |
| Unable to find a default server with ADWS | Máy chưa là DC. Làm lại mục A, hoặc chạy `Start-Service ADWS` |
| Join miền báo không tìm thấy miền | Đặt DNS của Client trỏ về IP của DC |
| Không đăng nhập được bằng phungcp | Kiểm tra user Enabled và Client đã vào miền |
| Tạo user báo mật khẩu không đạt | Dùng mật khẩu có chữ hoa, thường, số, ký tự đặc biệt |
