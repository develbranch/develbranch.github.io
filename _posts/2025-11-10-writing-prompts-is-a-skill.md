---
layout: post
title: "Kỹ năng viết prompt"
date: 2025-11-10 00:00:00.000000000 +07:00
type: post
published: true
status: publish
categories:
- Tutorials
tags:
- Level_1
- AI
- prompt
excerpt: 'Tôi muốn bắt đầu viết lại blog này bằng một chủ đề rất "hot" hiện nay, đó là sử dụng AI (Artificial Intelligence). Thay vì sử dụng AI theo cách đơn sơ, chỉ bằng cách viết prompt có cấu trúc, chúng ta có thể mở khoá một số tính năng cực kỳ tuyệt vời của AI.'
---

## Giới thiệu
Tôi gần đây có tham dự một số lớp học buổi tối sau giờ làm việc. Việc học mấy tiếng buổi tối quả thật rất tốn sức. Do đó, nếu một công cụ giúp tôi tóm tắt bài học để có thể tra lại sau khi học thì quả là tuyệt vời. Lúc này, tôi nghĩ ngay đến việc dùng AI để tóm tắt bài học, tra cứu video khi cần và cần biết chính xác thời điểm tương ứng với bài học để tôi có thể seek đến chính xác vị trí và xem chính xác nội dung cần tham khảo.


## Cần chuẩn bị những gì?
Để làm một "task" liên quan tới AI thì cần chuẩn bị 2 thứ:
 - 80% ở data engineering.
 - 20% ở viết prompt hợp lý.

## Data engineering
 - _Tải toàn bộ video về:_ Task này hoàn toàn không khó gì, kể cả với các video "private", cần một chút mẹo với trình duyệt là đủ.
 - _Tách audio từ video:_ task này cần một lệnh đơn giản với công cụ miễn phí, mã nguồn mở `ffmpeg`
 - _Chuyển đổi audio sang text:_ OpenAI đã release một model khá ổn cho việc này, có tên là [whisper](https://github.com/openai/whisper): Robust Speech Recognition via Large-Scale Weak Supervision. Điểm tuyệt vời của model này là miễn phí, hoàn toàn có thể chạy với các GPU phổ thông, cần VRAM khá ít (model large chỉ cần 10 GB VRAM). Vì video của tôi là bài giảng của một giảng viên nói tiếng Việt nên tôi chọn model này. Model hoạt động trên máy tính cá nhân của tôi (CPU Ryzen 3900X, 64GB RAM, GPU 4060Ti 16GB VRAM) với tốc độ khá tốt: transcript audio dài 3 giờ tốn khoảng 30 phút. Với ngôn ngữ tiếng Việt, tôi chọn model large (https://github.com/openai/whisper?tab=readme-ov-file#available-models-and-languages)
 
|  Size  | Parameters | English-only model | Multilingual model | Required VRAM | Relative speed |
|:------:|:----------:|:------------------:|:------------------:|:-------------:|:--------------:|
|  tiny  |    39 M    |     `tiny.en`      |       `tiny`       |     ~1 GB     |      ~10x      |
|  base  |    74 M    |     `base.en`      |       `base`       |     ~1 GB     |      ~7x       |
| small  |   244 M    |     `small.en`     |      `small`       |     ~2 GB     |      ~4x       |
| medium |   769 M    |    `medium.en`     |      `medium`      |     ~5 GB     |      ~2x       |
| large  |   1550 M   |        N/A         |      `large`       |    ~10 GB     |       1x       |
| turbo  |   809 M    |        N/A         |      `turbo`       |     ~6 GB     |      ~8x       |


Với các transcript tiếng Anh, hoàn toàn có thể chọn model có kích thước nhỏ hơn (medium hoặc small) để tăng tốc độ.

Sau bước này, chúng ta sẽ có một file text chứa toàn bộ transcript của video.


## Viết prompt

Đây là 20% còn lại của task. Hầu hết chúng ta đang kỳ vọng quá nhiều vào AI, mong muốn nó phải biết hết và đưa cho mình một câu trả lời toàn diện, có chiều sâu. Đáp lại kỳ vọng này lại là một kết quả chung chung, đọc qua thì có vẻ "hợp lý", nhưng thực thế lại thiếu chiều sâu và không có nhiều tác dụng.

Đối với tôi, model AI nào không quan trọng, vì khi dùng prompt đúng thì kết quả đầu ra tương đương nhau. Tuy nhiên, tôi khá thích các model mới như Claude sonnet 4.5 (Anthropic), ChatGPT-5 (OpenAI). Gemini 2.5 Pro (Google) cho kết quả cũng không tệ. Nếu đổi sang dùng DeepSeek thì chất lượng hơi giảm so với đánh giá của tôi, nhưng tôi cho rằng chấp nhận được. Một điểm quan tâm của tôi là giá thành. Các model AI cho kết quả tương đương thì chúng ta cứ chọn loại rẻ thôi.

Vấn đề chính nằm ở chỗ _"chúng ta đặt câu hỏi cho AI thế nào?"_ . Mặc dù AI không thể thay thế con người, nhưng chỉ bằng cách tối ưu prompt cho AI, chúng ta sẽ giúp AI trả lời tốt hơn.

Chúng ta có thể sử dụng công thức sau cho việc viết prompt cho AI, tạm gọi là công thức C.O.R.G.I.C.E:

- **Context (Bối cảnh)**: Nói rõ đầu vào, hoàn cảnh, ngôn ngữ, cấu trúc dữ liệu sẵn có.
- **Output format**: Khuôn mẫu đầu ra
- **Role (Vai trò)**: Nhập vai để xác định phong cách. Đây là một phương pháp cho phép AI tương tác và tạo câu trả lời đúng với yêu cầu định ra.
- **Goal (Mục tiêu)**: Kết quả cần đạt, giá trị cốt lõi của câu trả lời.
- **Instruction/Requirements (Hướng dẫn)**: Đây là phần hướng dẫn chi tiết cách làm cụ thể cách làm cụ thể để AI biết cần làm từng bước nào.
- **Constraints (Ràng buộc)**: Ràng buộc cứng cần tuân thủ. Đây là các "luật cấm" và ưu tiên định dạng. Có thể thêm các luật mô tả để AI chống "bịa" trong câu trả lời ("Do NOT invent", "No hallucinations", "No Yapping",...)
- **Examples (Ví dụ)**: minh họa để mô hình hiểu định dạng của dữ liệu.

### Ví dụ cụ thể
Quay trở lại vấn đề của tôi, từ công thức viết prompt trên, chúng ta có thể xây dựng một yêu cầu đầy đủ cho AI thực hiện như sau:

***Context (Bối cảnh)***

You are given a full transcript of an online lecture converted from video or audio.
Each line follows this structure:

`[START -> END] Utterance text`


Example transcript lines:
```
[8.44s -> 9.94s]  Chắc mọi người nghe được rồi.
[10.64s -> 16.04s]  Đã 8 giờ thì chắc là tôi xin phép là bắt đầu cái buổi học luôn.
```

Additional Context:
- Instructor: Instructor’s name
- Transcript file: path_to_transcript.txt
- Note: Replace “Executive Summary” with “Tóm tắt các ý trọng tâm”.

***Output Format***

Return the final result only in the following Markdown structure:
```
# [Lesson Title]  
- Instructor: [name or “not stated”]  
- Session date/time: [if stated] • Total speaking time (approx.): ~HH:MM:SS  
- Main objective(s): …  
- Prerequisites mentioned: …  

## Tóm tắt các ý trọng tâm (Executive Summary)
- 5–8 bullet points capturing the main takeaways.

## Table of Contents by Time
1. [hh:mm–hh:mm] Topic/Section — one-line gist  
2. [hh:mm–hh:mm] Topic/Section — one-line gist  
…

## Detailed Outline (time-anchored)
### [hh:mm–hh:mm] Section Title
- Key ideas:  
  - …
  - …
- Important definitions/frameworks:  
  - Term — short definition.
- Examples/demos/case studies:  
  - …
- Teacher’s guidance (tips, cautions, rules of thumb):  
  - …
- If relevant: equations/steps/checklists shown or described.
- Slide cues (if explicitly referenced): “Slide mentions …” [no fabrication]

## Methods / Systems Explained
- Name (e.g., “The Turtle system”): purpose, components, when/why to use, pros/cons, constraints, assumptions.

## Pros & Cons / Comparisons (if discussed)
- …

## Q&A / Misconceptions Addressed
- Question → concise answer [timestamp].  
- …

## Assignments / Action Items / Next Steps
- Practice items, deadlines, submissions, or readings.

## Resources Mentioned
- Books, articles, websites, datasets, code, or slide references (only if mentioned in transcript).

## Glossary of Terms (brief)
- Term: short definition.

## Key Quotes (optional, ≤5)
- “[translated quote]” — [timestamp]
```
***Role (Vai trò)***

You are an expert lecture summarizer.
Your expertise lies in producing faithful, structured, timestamp-anchored summaries of academic or professional lectures.

***Goal (Mục tiêu)***

Create a complete, accurate, and time-organized summary that helps learners quickly review the lesson content without hallucination or redundancy. Your summary should:
- Capture definitions, frameworks, formulas, examples, teacher’s advice, and cautions.
- Structure content by topics/segments with clear timestamps.
- Be concise but comprehensive, and written in clear, natural Vietnamese.

***Instructions (Hướng dẫn cụ thể)***

1) Use only transcript content. Do NOT invent slides, examples, or data not present in the text.

2) If unclear, write [unclear] instead of guessing.

3) Translate any quoted phrases into natural Vietnamese.

4) Remove filler talk (greetings, ums, repetition). Keep logistics only if they include instructions, deadlines, or assignments.

5) Merge short lines that share the same topic into a single summarized segment.

6) Normalize all times to the format hh:mm:ss.

7) Anchor each topic section with timestamps ([hh:mm–hh:mm]).

8) Use bullet points instead of long paragraphs.

9) Avoid repeating ideas or writing generic comments.

10) Respect the Markdown format strictly.


***Constraints (Ràng buộc)***
- Faithfulness: No hallucinations or assumptions. Everything must come directly from the transcript.
- Language: Final output must be written in Vietnamese.
- Format: Return only the Markdown summary.
- Completeness: Ensure all important parts of the lecture (definitions, examples, teacher’s comments, assignments) are covered.
- Clarity: Prefer brevity, logical order, and topic segmentation.

***Examples (Ví dụ minh họa)***

Input snippet:
```
[8.44s -> 9.94s]  Chắc mọi người nghe được rồi.
[10.64s -> 16.04s]  Đã 8 giờ thì chắc là tôi xin phép là bắt đầu cái buổi học luôn.
[16.10s -> 22.45s]  Hôm nay chúng ta sẽ học về khái niệm “marketing funnel” và cách ứng dụng nó trong chiến lược nội dung.
```

Output snippet (Markdown):
```
# Marketing Funnel Overview  
- Instructor: Not stated  
- Session date/time: Not stated • Total speaking time: ~00:22:45  
- Main objectives: Understand the concept of the marketing funnel and its application in content strategy.

## Tóm tắt các ý trọng tâm
- The class begins with an overview of “marketing funnel.”  
- The instructor explains its stages and relevance to content planning.  
- Students are guided to apply the concept in practical scenarios.


✅ Final Reminder:
Return only the formatted Markdown summary as specified.
Do not include explanations, raw transcript, or additional commentary.
```


## Tóm tắt một video

Bước này có khá nhiều cách để làm:
- Nếu bạn có sẵn tài khoản ChatGPT-plus trở lên và muốn dùng AI này, thì có thể tạo một [custom gpt](https://help.openai.com/en/articles/8554397-creating-a-gpt). System prompt chính là đoạn prompt trên.
- Trường hợp chúng ta muốn sử dụng ChatGPT bản thường (free) trở lên, không muốn tạo custom gpt, có thể lưu prompt phía trên thành một file, đặt tên là `instructions.md` và transcript sẽ lưu thành file `transcript.txt`. Với 2 file này, ChatGPT sẽ biết phải xử lý thế nào.
- Nếu đã quen với vscode, chúng ta hoàn toàn có thể thêm vào vscode một chế độ mới, để có thể thực hiện công việc này.
- Trường hợp chúng ta muốn mọi thứ trông giống lập trình viên hơn, có thể dùng đoạn code sau:
```
# Dependencies: openai python-dotenv

import os
import argparse
from openai import OpenAI
from dotenv import load_dotenv
from datetime import datetime

# Load environment variables from .env file
load_dotenv()

selected_model = "openai/gpt-5"
input_token_price = 1.5/1000  # Example price per 1K tokens for the selected model
output_token_price = 10/1000  # Example price per 1K tokens for the selected model

system_prompt = open("system_prompt.txt", "r", encoding='utf-8').read()

def summarize_text(client, text_content, model=selected_model, temperature=0.7):
    """
    Summarizes the given text using the specified model.
    """

    try:
        response = client.chat.completions.create(
            model=model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": text_content}
            ],
            temperature=temperature,
        )
        summary = response.choices[0].message.content
        usage = response.usage
        return summary, usage
    except Exception as e:
        print(f"An error occurred: {e}")
        return None, None

def main():
    parser = argparse.ArgumentParser(description="Summarize text file using AI.")
    parser.add_argument("input_file", type=str, help="Path to the transcript .txt file.")
    parser.add_argument("output_file", type=str, help="Path to the output file.")
    args = parser.parse_args()

    input_file = args.input_file
    output_file = args.output_file

    # It's recommended to set the API key in an environment variable `API_KEY`
    api_key = os.getenv("API_KEY")
    if not api_key:
        print("Error: API_KEY environment variable not set.")
        print("Please create a .env file and add API_KEY='your_key_here'")
        return
    base_url = os.getenv("BASE_URL", "https://openrouter.ai/api/v1") # set OpenAI key/openrouter

    client = OpenAI(
        base_url=base_url,
        api_key=api_key,
    )
    with open(input_file, 'r', encoding='utf-8') as f:
        print(f"Processing file: {input_file}")
        content = f.read()
        summary, usage = summarize_text(client, content)
        if summary and usage:
            with open(output_file, 'w', encoding='utf-8') as f:
                f.write(summary)
                print(f"Successfully summarized and saved to {output_file}")
            # Calculate cost (example pricing, replace with actual if known)
            prompt_cost = (usage.prompt_tokens / 1000) * input_token_price
            completion_cost = (usage.completion_tokens / 1000) * output_token_price
            total_cost = prompt_cost + completion_cost
            print(f"{datetime.now().strftime('%y-%m-%d %a %H:%M:%S')} File: {input_file}, Tokens: {usage.total_tokens}, Cost: ${total_cost:.6f}\n")


if __name__ == "__main__":
    main()
```


## Thực hành với một video bất kỳ

Tôi chọn ngẫu nhiên một video nào đó: [Impostor Syndrome - Hacking Apple MDMs Using Rogue Device Enrolments](https://www.youtube.com/watch?v=qFxBneMlYZQ). Đây là bài trình bày của 
kỹ sư bảo mật [Marcell Molnár](https://www.linkedin.com/in/marcell-molnar/).

Sau khi tải transcript về (rất tuyệt là youtube có luôn tính năng này), tôi thử tóm tắt toàn bộ video, bằng Claude sonnet 4.5, và kết quả khá mĩ mãn.

```
Context Length	30.2k / 1.0m
Tokens	↑ 66.4k  ↓ 6.8k
Cache	36.8k
API Cost	$0.22
Size	61.7 kB
```

## Đây là kết quả


### Bảo mật macOS MDM: Lỗ hổng enrollment dựa trên serial number

  

- Giảng viên: Marcell Molnár

- Thời gian phát biểu (ước tính): ~34:48

- Mục tiêu chính: Trình bày về lỗ hổng bảo mật nghiêm trọng trong quy trình MDM enrollment của macOS, cho phép tấn công dựa trên serial number của thiết bị

- Kiến thức tiên quyết: Hiểu biết cơ bản về MDM (Mobile Device Management), macOS, reverse engineering

  

#### Tóm tắt các ý trọng tâm

  

- Lỗ hổng cho phép attacker giả mạo thiết bị Apple chỉ bằng cách sử dụng serial number (10-12 ký tự)

- Apple Business Manager và MDM enrollment chỉ xác thực dựa trên serial number, không có cơ chế bảo mật bổ sung

- Khoảng 60% trường hợp không có SSO, cho phép enrollment trực tiếp

- Có thể thu thập thông tin nhạy cảm: Wi-Fi passwords, credentials, API keys, và thậm chí root access trên hàng chục nghìn thiết bị

- Apple đã biết về vấn đề này ít nhất 8 năm (từ bài báo của Duo Security)

- Các biện pháp phòng ngừa chính: bật SSO, kiểm soát scope của credentials, không embed API keys trong shell scripts

  

#### Mục lục theo thời gian

  

1.  **[00:03-02:52]** Giới thiệu và động lực nghiên cứu

2.  **[02:52-06:04]** Phát hiện ban đầu: Phantom device từ Brazil

3.  **[06:04-09:32]** Tái tạo lỗ hổng với hackintosh VM

4.  **[09:32-13:32]** Reverse engineering quy trình MDM

5.  **[13:32-17:10]** Weaponization và bypass techniques

6.  **[17:10-23:28]** Phân loại "prizes" (thông tin thu được)

7.  **[23:28-25:48]** Jackpot: Root access thông qua MDM API keys

8.  **[25:48-30:01]** Biện pháp phòng ngừa và khuyến nghị

9.  **[30:01-34:48]** Q&A session

  

#### Chi tiết bài giảng (theo thời gian)

  

##### [00:03-02:52] Giới thiệu và bối cảnh

  

-  **Ý tưởng chính:**

- So sánh lỗ hổng như một "slot machine" - spin và có thể win các thông tin có giá trị

- Giảng viên từ Hungary, làm việc tại Form 3 (công ty fintech UK)

- Jackpot lớn nhất: root access trên 45,000 thiết bị cùng lúc

  

-  **Cấu trúc bài thuyết trình:**

1. Journey và động lực nghiên cứu

2. Hardcore reverse engineering

3. Prizes và exploitation

  

##### [02:52-06:04] Phát hiện ban đầu: Phantom device alert

  

-  **Ý tưởng chính:**

- Alert từ SOCK: một thiết bị đăng nhập từ Brazil

- Thiết bị gốc vẫn online ở Germany

- Xuất hiện "phantom device" - bản duplicate trong MDM

- Điểm chung duy nhất: serial number

  

-  **Khái niệm quan trọng:**

-  **Apple Business Manager (ABM):** Registry tự động cho các thiết bị Apple mua từ Apple hoặc authorized reseller

-  **Zero-touch enrollment:** Thiết bị tự động enroll vào MDM khi unbox lần đầu, không cần cài đặt thủ công

  

-  **Phát hiện then chốt:**

- MDM enrollment chỉ phụ thuộc vào serial number

- Serial number không phải thông tin bí mật (in trên hộp, có trong manifests, GitHub)

  

##### [06:04-09:32] Tái tạo lỗ hổng

  

-  **Công cụ sử dụng:**

- OSX-KVM project (Kolia trên GitHub) - tạo macOS VM trên Ubuntu

- OpenCore - boot disc compiler

  

-  **Quy trình thử nghiệm:**

1. Tạo VM macOS

2. Edit config với serial number của thiết bị bị compromise

3. Recompile OpenCore

4. Boot VM → tự động enroll vào MDM

  

-  **Kết quả:**

- Thành công tạo thêm phantom device

- Xác nhận: chỉ cần serial number để enroll

- May mắn: MDM chỉ có Adobe Acrobat free version trong self-service

  

-  **Insight:**

- Nhiều công ty có thể deploy thông tin nhạy cảm hơn qua MDM

- Cần nghiên cứu sâu hơn về mức độ rủi ro

  

##### [09:32-13:32] Reverse engineering quy trình MDM

  

-  **Cấu trúc serial number (format cũ, đến 2021):**

- 10-12 ký tự alphanumeric

- Format cố định: `[Location][Year][Week][Sequential Number][Model]`

- Manufacturing location: chỉ 2 variants

- Rất dễ generate serial numbers hợp lệ

  

-  **Ba thành phần chính trong MDM process:**

1. Apple device (mới unbox)

2. iprofiles.apple.com (Apple Business Manager API)

3. Company's MDM server

  

-  **Message flow:**

  

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

  

-  **Tài liệu tham khảo quan trọng:**

- "MDM Me Maybe" article của Duo Security (8 năm trước)

- Là nguồn thông tin duy nhất về process này trên internet

  

##### [13:32-17:10] Weaponization và bypass kỹ thuật

  

-  **Chiến lược weaponization:**

1. Generate hoặc scrape serial numbers từ GitHub

2. Poll Apple API

3. Thu thập MDM server addresses

4. Thử enroll

  

-  **Kỹ thuật instrumentation:**

- Sử dụng LLDB để instrument profiles command

- Swap serial number trong memory

- Capture messages và MDM addresses

  

-  **Roadblock 1: Client-side rate limiting**

- Vấn đề: Script dừng sau 10 serial numbers

- Nguyên nhân: Profile enrollment dates được lưu trong file, giới hạn 10 dòng

-  **Bypass:**  `rm /var/db/ConfigurationProfiles/Settings/.profilesAreInstalled` (client-side validation!)

- Edge case: nhớ tăng year khi chuyển năm mới

  

-  **Roadblock 2: SSO protection**

-  **Bypass technique:** Exploit endpoint discrepancy

- XML response có thể chứa 2 URLs khác nhau (new & legacy)

- Edit enrollment profile on disk, swap URL mới → URL cũ

- Legacy endpoint có thể không có SSO (~60% success rate)

  

-  **Lưu ý kỹ thuật:**

- Configuration có thể hidden trong MDM settings

- Admins có thể quên secure legacy endpoint khi migrate

  

##### [17:10-23:28] Phân loại "prizes" - Tier system

  

###### **Tier 1: Basic information (luôn có)**

  

-  **Company information (không cần bypass SSO):**

- Support phone number

- Email addresses

- Person phụ trách MDM

- Office locations

- MDM server address

  

-  **Configuration profiles (nếu bypass SSO hoặc không SSO):**

- Password policies

- EDR agent info

- VPN certificates

- Network passwords

- Xảy ra với mọi enrollment

  

-  **Self-service applications:**

- Free software (thấp giá trị)

- Licensed software (Microsoft Office, etc.)

-  **Internal tools** - mở attack surface mới

- Custom company-developed tools

  

-  **Local administrator password:**

- Khi MDM agent install → tạo local account

- Nếu admin sloppy: same password cho tất cả devices

- DSCL transfers password in **clear text** → check Burp logs

- Có thể reuse across all machines

  

###### **Tier 2: Cooler prizes (cần kỹ thuật nâng cao)**

  

-  **Wi-Fi passwords:**

-  **Vấn đề:** VM không có wireless card → Apple từ chối install wireless profiles

-  **Giải pháp:** Reverse engineer profile installation process

- Profiles được encrypted & signed với per-device certificate

-  **Bypass:** Hook NS dictionary function với LLDB

- Dump password from memory trước khi rejection

- Bug bounty: $50 cho network password từ California :)

  

-  **Credentials trong shell scripts:**

- Admins giải quyết problems ngoài MDM bằng custom shell scripts

-  **Tìm thấy:** Slack tokens, GitHub credentials, SharePoint API keys, AD/LDAP passwords

-  **Root cause:** Misconception về "closed ecosystem"

- Reality: Rogue enrollment → unsafe environment → credentials exposed

  

-  **Ví dụ về shell script abuse:**

- MDM agent runs as root

- Small mistakes → race conditions

- Download to /tmp or risky copy operations

- Users có thể trigger từ self-service

- Reliable local privilege escalation

  

##### [23:28-25:48] JACKPOT: Root access via MDM API keys

  

-  **Kịch bản điển hình:**

- Admin hit brick wall với MDM capabilities

- Can't solve với shell scripts alone

- Vendor FAQ suggests: "Put API key in shell script for MDM API access"

  

-  **Vulnerability chain:**

1. API key embedded trong shell script

2. Shell script pushed qua MDM

3. Rogue enrollment retrieves script

4. Attacker obtains API key

  

-  **MDM API abuse:**

- Shout-out to co-author Magdalina

- Mỗi MDM API có "reverse shell function" ẩn

- Cho phép push one-liner shell scripts đến bất kỳ enrolled device nào

-  **Result:** Root access trên tất cả devices

  

-  **Vấn đề về scope:**

- Admins đôi khi restrict token scope

- Nhưng: Black box nature → không nhận ra risk

- Ngay cả read-only access cũng leak sensitive info (PII)

  

-  **Impact thực tế:**

- Giảng viên đạt: root access trên 45,000 devices simultaneously

  

##### [25:48-30:01] Biện pháp phòng ngừa và khuyến nghị

  

-  **Defense #0: Bật SSO (must-have)**

- Luôn enable SSO cho MDM enrollment

- Kiểm tra documentation cho legacy endpoints

- Ensure SSO extends to cả hai endpoints (new & legacy)

- Legacy endpoint có thể dùng LDAP-based SSO thay vì "true SSO"

-  **Rule:** Turn on ALL checkboxes

  

-  **Defense #1: Credential management trong shell scripts**

-  **Mindset:** Luôn assume credentials sẽ落vào tay users hoặc attackers

- Think twice trước khi push

- Reduce scope tối đa

- Apply least privilege principle

- Security review bởi in-house team

  

-  **Defense #2: Admin password policies**

- Sử dụng per-machine passwords thay vì single shared password

- Tránh lateral movement từ red team

- Remote attacker risk thấp hơn nhưng internal threat cao

  

-  **Defense #3: SSO cho self-service (separately)**

- Last line of defense

- Nhiều abusable tools không phải trong profiles mà trong self-service

- Highly recommended

  

-  **Defense #4: Local privilege escalation prevention**

- Shell scripts running as root → high risk

- Common mistakes:

- Download to /tmp

- Risky copy operations

- Race conditions → reliable local root exploit

- Users trigger từ self-service

- Red team có thể easily abuse

  

-  **Defense #5: MDM API key protection**

-  **NEVER** put MDM's own API key trong shell scripts

- Nếu absolutely necessary:

- Double-check scope restrictions

- Verify không có abusable functions

- Download vendor's Postman collection

- Test key capabilities yourself

- Ngay cả read-only access exposes PII

  

##### [30:01-34:48] Q&A Session

  

-  **Q1: Legacy vs new endpoint differences?**

- A: Exact same end result và enrollment

- Legacy primarily cho mobile/iPhone, new cho MacBooks

- Vendor independent (per Apple MDM developer documentation)

  

-  **Q2: Chỉ cần serial number để enroll?**

- A: Đúng, không authentication khác

- Duo Security reported 8 năm trước

- Apple's response: không provide thêm security, shift responsibility to vendor

- "Phone book approach" - lookup MDM by serial number

- Apple obfuscates process (different TCP protocol) nhưng server-side vẫn simple

  

-  **Q3: MDM protocol vulnerabilities?**

- A: Next research step

- Plan: tạo scanner cho administrators

- Đã report vulnerabilities cho MDM vendors (chưa disclose)

-  **Gap:** Apple assumes MDM vendor provides security, vendors assume Apple vets requests

- Falls through the cracks

  

#### Phương pháp / Hệ thống được giải thích

  

##### Apple Business Manager + MDM Enrollment Flow

  

-  **Mục đích:** Zero-touch device management cho organizations

-  **Components:**

1. Apple Business Manager (device registry)

2. Company MDM server (on-prem hoặc SaaS)

3. iprofiles.apple.com (Apple API)

4. Device serial number (authentication token)

  

-  **Workflow:**

1. Company đăng ký ABM

2. Devices tự động register khi mua

3. Configure MDM server address trong ABM

4. Device unbox → tự động connect → enroll

5. Company logo hiện ngay khi boot

  

-  **Ưu điểm:** User-friendly, automatic, không cần manual setup

-  **Nhược điểm:** Security phụ thuộc hoàn toàn vào serial number secrecy

-  **Assumptions:** Serial number là protected information (sai!)

-  **Constraints:** Apple coi đây là phone book service, không thêm security layer

  

##### Serial Number Format (Pre-2021)

  

-  **Structure:**  `[Loc][YY][WW][XXXX][Model]`

- Loc: Manufacturing location (2 variants)

- YY: Year

- WW: Week

- XXXX: Sequential number

- Model: 4 chars model identifier

  

-  **Weaknesses:**

- Rất predictable

- Small key space

- Dễ generate valid numbers

- Public information (boxes, GitHub, manifests)

  

###### So sánh / Ưu nhược điểm

  

##### MDM Endpoint Comparison

  

| Aspect | New (Web-based) | Legacy (XML-based) |

|--------|-----------------|-------------------|

| Format | B64 encoded | XML |

| Primary use | MacBooks | Mobile/iPhone |

| SSO | Thường có | Có thể bypass |

| Security | LDAP hoặc modern SSO | Often LDAP-only |

| Risk | Medium | Higher (forgotten) |

  

##### Scope Restriction Effectiveness

  

**Pros của restricted scope:**

- Giảm blast radius

- Limit damage potential

  

**Cons/Limitations:**

- Admins không realize ease of extraction

- Black box nature → false sense of security

- Read-only vẫn leak PII

- Attackers có thể chain với other vulns

  

#### Q&A / Hiểu lầm đã được làm rõ

  

-  **Q:** Có cần Apple account để enroll không?

-  **A:** Không, chỉ cần serial number. Apple không provide additional security, shift responsibility to vendors [32:13-33:26]

  

-  **Q:** Legacy và new endpoint có giống nhau không?

-  **A:** Exactly the same enrollment result. Chỉ khác format (XML vs B64). Vendor independent [31:09-32:06]

  

-  **Q:** Apple có biết issue này không?

-  **A:** Có, ít nhất 8 năm. Duo Security reported, Apple declined to fix, coi như feature not bug [32:33-33:19]

  

-  **Hiểu lầm phổ biến:** MDM là "closed ecosystem" an toàn

-  **Reality:** Rogue enrollment dễ dàng expose mọi thứ kể cả credentials trong scripts

  

#### Assignments / Hành động tiếp theo

  

Dành cho MDM Administrators:

1.  **Immediate actions:**

- Audit SSO configuration (both endpoints)

- Review shell scripts cho embedded credentials

- Check API key scopes trong scripts

- Enable SSO cho self-service

  

2.  **Short-term:**

- Implement per-device admin passwords

- Security review all custom scripts

- Download vendor Postman collection, test API keys

- Document endpoint configurations

  

3.  **Long-term:**

- Establish script review process

- Regular security audits

- Monitor MDM vendor security advisories

- Consider scanner tools (upcoming from researcher)

  

Dành cho Security Researchers:

- Investigate MDM protocol vulnerabilities

- Test vendor implementations

- Develop automated scanning tools

  

#### Tài nguyên được đề cập

  

-  **Duo Security - "MDM Me Maybe" article** (8 years old, duy nhất source về MDM process)

-  **OSX-KVM project** by Kolia (GitHub) - MacOS VM trên Ubuntu

-  **OpenCore** - Boot disc compiler cho hackintosh

-  **Apple MDM Developer Documentation** - Vendor implementation guide

-  **Open source MDM project** (GitHub) - Reference implementation

-  **Vendor Postman collections** - API testing

  

#### Thuật ngữ chính

  

-  **MDM (Mobile Device Management):** Hệ thống quản lý thiết bị từ xa, cho phép deploy configs, apps, và monitor devices

-  **Apple Business Manager (ABM):** Portal đăng ký devices của Apple cho organizations, tự động registry khi purchase

-  **Zero-touch enrollment:** Quy trình tự động enroll device vào MDM khi unbox lần đầu, không cần user intervention

-  **iprofiles.apple.com:** Apple API endpoint cho MDM enrollment lookups

-  **Serial number:** Unique identifier 10-12 chars cho mỗi Apple device, FORMAT PREDICTABLE, không cryptographically secure

-  **SSO (Single Sign-On):** Authentication layer có thể thêm vào MDM enrollment để prevent unauthorized access

-  **Self-service:** Application drawer trong MDM cho phép users tự install approved software

-  **Shell script abuse:** Technique exploit MDM shell script execution với root privileges

-  **DSCL (Directory Service Command Line):** macOS utility quản lý user accounts, transfers passwords in clear text

-  **LLDB:** Debugger sử dụng để hook và intercept function calls, extract data from memory

-  **Phantom device:** Rogue enrolled device với duplicate serial number nhưng different hardware

-  **Legacy endpoint:** XML-based MDM enrollment endpoint, thường bị quên secure khi migrate to new endpoint

-  **API key scope:** Permissions granted to API key, should be restricted nhưng thường overly permissive

  

#### Trích dẫn quan trọng

  

- "[00:49] The vulnerability that I'm going to talk about today is kind of like a slot machine. Just sit in front of your computer, give it a spin, and you can win stuff."

  

- "[01:31] The biggest jackpot that I've ever managed to hit with this vulnerability was root access on 45,000 devices at the same time."

  

- "[08:28] The whole enrollment just depends on the serial number itself which is a bit scary if you think about it."

  

- "[14:36] Apple has known this issue for at least eight years." (referring to Duo Security article)

  

- "[29:23] Please do not put the MDM's own API key into any of your shell scripts. Just don't."




## Kết luận

Qua bài viết vừa rồi, tôi hy vọng rằng các bạn có thể có cái nhìn sâu sắc hơn với cách viết prompt cho AI. AI rất thông minh, nhưng nếu dùng đúng prompt thì AI vừa thông minh vừa được việc. Chúc các bạn thành công./.