
# Bảo mật macOS MDM: Lỗ hổng enrollment dựa trên serial number

- Giảng viên: Marcell Molnár
- Thời gian phát biểu (ước tính): ~34:48
- Mục tiêu chính: Trình bày về lỗ hổng bảo mật nghiêm trọng trong quy trình MDM enrollment của macOS, cho phép tấn công dựa trên serial number của thiết bị
- Kiến thức tiên quyết: Hiểu biết cơ bản về MDM (Mobile Device Management), macOS, reverse engineering

## Tóm tắt các ý trọng tâm

- Lỗ hổng cho phép attacker giả mạo thiết bị Apple chỉ bằng cách sử dụng serial number (10-12 ký tự)
- Apple Business Manager và MDM enrollment chỉ xác thực dựa trên serial number, không có cơ chế bảo mật bổ sung
- Khoảng 60% trường hợp không có SSO, cho phép enrollment trực tiếp
- Có thể thu thập thông tin nhạy cảm: Wi-Fi passwords, credentials, API keys, và thậm chí root access trên hàng chục nghìn thiết bị
- Apple đã biết về vấn đề này ít nhất 8 năm (từ bài báo của Duo Security)
- Các biện pháp phòng ngừa chính: bật SSO, kiểm soát scope của credentials, không embed API keys trong shell scripts

## Mục lục theo thời gian

1. **[00:03-02:52]** Giới thiệu và động lực nghiên cứu
2. **[02:52-06:04]** Phát hiện ban đầu: Phantom device từ Brazil
3. **[06:04-09:32]** Tái tạo lỗ hổng với hackintosh VM
4. **[09:32-13:32]** Reverse engineering quy trình MDM
5. **[13:32-17:10]** Weaponization và bypass techniques
6. **[17:10-23:28]** Phân loại "prizes" (thông tin thu được)
7. **[23:28-25:48]** Jackpot: Root access thông qua MDM API keys
8. **[25:48-30:01]** Biện pháp phòng ngừa và khuyến nghị
9. **[30:01-34:48]** Q&A session

## Chi tiết bài giảng (theo thời gian)

### [00:03-02:52] Giới thiệu và bối cảnh

- **Ý tưởng chính:**
  - So sánh lỗ hổng như một "slot machine" - spin và có thể win các thông tin có giá trị
  - Giảng viên từ Hungary, làm việc tại Form 3 (công ty fintech UK)
  - Jackpot lớn nhất: root access trên 45,000 thiết bị cùng lúc

- **Cấu trúc bài thuyết trình:**
  1. Journey và động lực nghiên cứu
  2. Hardcore reverse engineering
  3. Prizes và exploitation

### [02:52-06:04] Phát hiện ban đầu: Phantom device alert

- **Ý tưởng chính:**
  - Alert từ SOCK: một thiết bị đăng nhập từ Brazil
  - Thiết bị gốc vẫn online ở Germany
  - Xuất hiện "phantom device" - bản duplicate trong MDM
  - Điểm chung duy nhất: serial number

- **Khái niệm quan trọng:**
  - **Apple Business Manager (ABM):** Registry tự động cho các thiết bị Apple mua từ Apple hoặc authorized reseller
  - **Zero-touch enrollment:** Thiết bị tự động enroll vào MDM khi unbox lần đầu, không cần cài đặt thủ công

- **Phát hiện then chốt:**
  - MDM enrollment chỉ phụ thuộc vào serial number
  - Serial number không phải thông tin bí mật (in trên hộp, có trong manifests, GitHub)

### [06:04-09:32] Tái tạo lỗ hổng

- **Công cụ sử dụng:**
  - OSX-KVM project (Kolia trên GitHub) - tạo macOS VM trên Ubuntu
  - OpenCore - boot disc compiler

- **Quy trình thử nghiệm:**
  1. Tạo VM macOS
  2. Edit config với serial number của thiết bị bị compromise
  3. Recompile OpenCore
  4. Boot VM → tự động enroll vào MDM

- **Kết quả:**
  - Thành công tạo thêm phantom device
  - Xác nhận: chỉ cần serial number để enroll
  - May mắn: MDM chỉ có Adobe Acrobat free version trong self-service

- **Insight:**
  - Nhiều công ty có thể deploy thông tin nhạy cảm hơn qua MDM
  - Cần nghiên cứu sâu hơn về mức độ rủi ro

### [09:32-13:32] Reverse engineering quy trình MDM

- **Cấu trúc serial number (format cũ, đến 2021):**
  - 10-12 ký tự alphanumeric
  - Format cố định: `[Location][Year][Week][Sequential Number][Model]`
  - Manufacturing location: chỉ 2 variants
  - Rất dễ generate serial numbers hợp lệ

- **Ba thành phần chính trong MDM process:**
  1. Apple device (mới unbox)
  2. iprofiles.apple.com (Apple Business Manager API)
  3. Company's MDM server

- **Message flow:**

  **Bước 1: Device → Apple Business Manager**
  - Gửi JSON nhỏ chỉ chứa serial number
  - Response:
    - Empty nếu device không enroll
    - Dictionary lớn với thông tin công ty nếu enrolled (support phone, email, MDM server address)

  **Bước 2: Device → MDM Server**
  - Hai endpoints khả dụng:
    - Web-based mới (B64 encoded payload)
    - XML-based cũ (same payload, XML format)
  - Payload chứa: hardware info, serial number, UID
  - Chỉ serial number được sử dụng thực sự

  **Bước 3: MDM Response**
  - Nếu có SSO: bị block (~40% trường hợp)
  - Không SSO: enrollment thành công (~60% trường hợp)

- **Tài liệu tham khảo quan trọng:**
  - "MDM Me Maybe" article của Duo Security (8 năm trước)
  - Là nguồn thông tin duy nhất về process này trên internet

### [13:32-17:10] Weaponization và bypass kỹ thuật

- **Chiến lược weaponization:**
  1. Generate hoặc scrape serial numbers từ GitHub
  2. Poll Apple API
  3. Thu thập MDM server addresses
  4. Thử enroll

- **Kỹ thuật instrumentation:**
  - Sử dụng LLDB để instrument profiles command
  - Swap serial number trong memory
  - Capture messages và MDM addresses

- **Roadblock 1: Client-side rate limiting**
  - Vấn đề: Script dừng sau 10 serial numbers
  - Nguyên nhân: Profile enrollment dates được lưu trong file, giới hạn 10 dòng
  - **Bypass:** `rm /var/db/ConfigurationProfiles/Settings/.profilesAreInstalled` (client-side validation!)
  - Edge case: nhớ tăng year khi chuyển năm mới

- **Roadblock 2: SSO protection**
  - **Bypass technique:** Exploit endpoint discrepancy
  - XML response có thể chứa 2 URLs khác nhau (new & legacy)
  - Edit enrollment profile on disk, swap URL mới → URL cũ
  - Legacy endpoint có thể không có SSO (~60% success rate)

- **Lưu ý kỹ thuật:**
  - Configuration có thể hidden trong MDM settings
  - Admins có thể quên secure legacy endpoint khi migrate

### [17:10-23:28] Phân loại "prizes" - Tier system

#### **Tier 1: Basic information (luôn có)**

- **Company information (không cần bypass SSO):**
  - Support phone number
  - Email addresses
  - Person phụ trách MDM
  - Office locations
  - MDM server address

- **Configuration profiles (nếu bypass SSO hoặc không SSO):**
  - Password policies
  - EDR agent info
  - VPN certificates
  - Network passwords
  - Xảy ra với mọi enrollment

- **Self-service applications:**
  - Free software (thấp giá trị)
  - Licensed software (Microsoft Office, etc.)
  - **Internal tools** - mở attack surface mới
  - Custom company-developed tools

- **Local administrator password:**
  - Khi MDM agent install → tạo local account
  - Nếu admin sloppy: same password cho tất cả devices
  - DSCL transfers password in **clear text** → check Burp logs
  - Có thể reuse across all machines

#### **Tier 2: Cooler prizes (cần kỹ thuật nâng cao)**

- **Wi-Fi passwords:**
  - **Vấn đề:** VM không có wireless card → Apple từ chối install wireless profiles
  - **Giải pháp:** Reverse engineer profile installation process
  - Profiles được encrypted & signed với per-device certificate
  - **Bypass:** Hook NS dictionary function với LLDB
  - Dump password from memory trước khi rejection
  - Bug bounty: $50 cho network password từ California :)

- **Credentials trong shell scripts:**
  - Admins giải quyết problems ngoài MDM bằng custom shell scripts
  - **Tìm thấy:** Slack tokens, GitHub credentials, SharePoint API keys, AD/LDAP passwords
  - **Root cause:** Misconception về "closed ecosystem"
  - Reality: Rogue enrollment → unsafe environment → credentials exposed

- **Ví dụ về shell script abuse:**
  - MDM agent runs as root
  - Small mistakes → race conditions
  - Download to /tmp or risky copy operations
  - Users có thể trigger từ self-service
  - Reliable local privilege escalation

### [23:28-25:48] JACKPOT: Root access via MDM API keys

- **Kịch bản điển hình:**
  - Admin hit brick wall với MDM capabilities
  - Can't solve với shell scripts alone
  - Vendor FAQ suggests: "Put API key in shell script for MDM API access"

- **Vulnerability chain:**
  1. API key embedded trong shell script
  2. Shell script pushed qua MDM
  3. Rogue enrollment retrieves script
  4. Attacker obtains API key

- **MDM API abuse:**
  - Shout-out to co-author Magdalina
  - Mỗi MDM API có "reverse shell function" ẩn
  - Cho phép push one-liner shell scripts đến bất kỳ enrolled device nào
  - **Result:** Root access trên tất cả devices

- **Vấn đề về scope:**
  - Admins đôi khi restrict token scope
  - Nhưng: Black box nature → không nhận ra risk
  - Ngay cả read-only access cũng leak sensitive info (PII)

- **Impact thực tế:**
  - Giảng viên đạt: root access trên 45,000 devices simultaneously

### [25:48-30:01] Biện pháp phòng ngừa và khuyến nghị

- **Defense #0: Bật SSO (must-have)**
  - Luôn enable SSO cho MDM enrollment
  - Kiểm tra documentation cho legacy endpoints
  - Ensure SSO extends to cả hai endpoints (new & legacy)
  - Legacy endpoint có thể dùng LDAP-based SSO thay vì "true SSO"
  - **Rule:** Turn on ALL checkboxes

- **Defense #1: Credential management trong shell scripts**
  - **Mindset:** Luôn assume credentials sẽ落vào tay users hoặc attackers
  - Think twice trước khi push
  - Reduce scope tối đa
  - Apply least privilege principle
  - Security review bởi in-house team

- **Defense #2: Admin password policies**
  - Sử dụng per-machine passwords thay vì single shared password
  - Tránh lateral movement từ red team
  - Remote attacker risk thấp hơn nhưng internal threat cao

- **Defense #3: SSO cho self-service (separately)**
  - Last line of defense
  - Nhiều abusable tools không phải trong profiles mà trong self-service
  - Highly recommended

- **Defense #4: Local privilege escalation prevention**
  - Shell scripts running as root → high risk
  - Common mistakes:
    - Download to /tmp
    - Risky copy operations
  - Race conditions → reliable local root exploit
  - Users trigger từ self-service
  - Red team có thể easily abuse

- **Defense #5: MDM API key protection**
  - **NEVER** put MDM's own API key trong shell scripts
  - Nếu absolutely necessary:
    - Double-check scope restrictions
    - Verify không có abusable functions
    - Download vendor's Postman collection
    - Test key capabilities yourself
  - Ngay cả read-only access exposes PII

### [30:01-34:48] Q&A Session

- **Q1: Legacy vs new endpoint differences?**
  - A: Exact same end result và enrollment
  - Legacy primarily cho mobile/iPhone, new cho MacBooks
  - Vendor independent (per Apple MDM developer documentation)

- **Q2: Chỉ cần serial number để enroll?**
  - A: Đúng, không authentication khác
  - Duo Security reported 8 năm trước
  - Apple's response: không provide thêm security, shift responsibility to vendor
  - "Phone book approach" - lookup MDM by serial number
  - Apple obfuscates process (different TCP protocol) nhưng server-side vẫn simple

- **Q3: MDM protocol vulnerabilities?**
  - A: Next research step
  - Plan: tạo scanner cho administrators
  - Đã report vulnerabilities cho MDM vendors (chưa disclose)
  - **Gap:** Apple assumes MDM vendor provides security, vendors assume Apple vets requests
  - Falls through the cracks

## Phương pháp / Hệ thống được giải thích

### Apple Business Manager + MDM Enrollment Flow

- **Mục đích:** Zero-touch device management cho organizations
- **Components:**
  1. Apple Business Manager (device registry)
  2. Company MDM server (on-prem hoặc SaaS)
  3. iprofiles.apple.com (Apple API)
  4. Device serial number (authentication token)

- **Workflow:**
  1. Company đăng ký ABM
  2. Devices tự động register khi mua
  3. Configure MDM server address trong ABM
  4. Device unbox → tự động connect → enroll
  5. Company logo hiện ngay khi boot

- **Ưu điểm:** User-friendly, automatic, không cần manual setup
- **Nhược điểm:** Security phụ thuộc hoàn toàn vào serial number secrecy
- **Assumptions:** Serial number là protected information (sai!)
- **Constraints:** Apple coi đây là phone book service, không thêm security layer

### Serial Number Format (Pre-2021)

- **Structure:** `[Loc][YY][WW][XXXX][Model]`
  - Loc: Manufacturing location (2 variants)
  - YY: Year
  - WW: Week
  - XXXX: Sequential number
  - Model: 4 chars model identifier

- **Weaknesses:**
  - Rất predictable
  - Small key space
  - Dễ generate valid numbers
  - Public information (boxes, GitHub, manifests)

## So sánh / Ưu nhược điểm

### MDM Endpoint Comparison

| Aspect | New (Web-based) | Legacy (XML-based) |
|--------|-----------------|-------------------|
| Format | B64 encoded | XML |
| Primary use | MacBooks | Mobile/iPhone |
| SSO | Thường có | Có thể bypass |
| Security | LDAP hoặc modern SSO | Often LDAP-only |
| Risk | Medium | Higher (forgotten) |

### Scope Restriction Effectiveness

**Pros của restricted scope:**
- Giảm blast radius
- Limit damage potential

**Cons/Limitations:**
- Admins không realize ease of extraction
- Black box nature → false sense of security
- Read-only vẫn leak PII
- Attackers có thể chain với other vulns

## Q&A / Hiểu lầm đã được làm rõ

- **Q:** Có cần Apple account để enroll không?
  - **A:** Không, chỉ cần serial number. Apple không provide additional security, shift responsibility to vendors [32:13-33:26]

- **Q:** Legacy và new endpoint có giống nhau không?
  - **A:** Exactly the same enrollment result. Chỉ khác format (XML vs B64). Vendor independent [31:09-32:06]

- **Q:** Apple có biết issue này không?
  - **A:** Có, ít nhất 8 năm. Duo Security reported, Apple declined to fix, coi như feature not bug [32:33-33:19]

- **Hiểu lầm phổ biến:** MDM là "closed ecosystem" an toàn
  - **Reality:** Rogue enrollment dễ dàng expose mọi thứ kể cả credentials trong scripts

## Assignments / Hành động tiếp theo

Dành cho MDM Administrators:
1. **Immediate actions:**
   - Audit SSO configuration (both endpoints)
   - Review shell scripts cho embedded credentials
   - Check API key scopes trong scripts
   - Enable SSO cho self-service

2. **Short-term:**
   - Implement per-device admin passwords
   - Security review all custom scripts
   - Download vendor Postman collection, test API keys
   - Document endpoint configurations

3. **Long-term:**
   - Establish script review process
   - Regular security audits
   - Monitor MDM vendor security advisories
   - Consider scanner tools (upcoming from researcher)

Dành cho Security Researchers:
- Investigate MDM protocol vulnerabilities
- Test vendor implementations
- Develop automated scanning tools

