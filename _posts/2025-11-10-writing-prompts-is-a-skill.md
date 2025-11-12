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
excerpt: 'Tôi muốn bắt đầu viết lại blog này bằng một chủ đề rất "hot" hiện nay, đó là sử dụng AI (Artificial Intelligence). Thay vì sử dụng AI theo cách đơn sơ, chỉ bằng cách viết prompt có cấu trúc, chúng ta có thể mở khoá một số tính năng cực kỳ tuyệt vời của AI. Đó gọi là Prompt Engineering.'
---

## Giới thiệu
Tôi gần đây có tham dự một số lớp học buổi tối sau giờ làm việc. Việc học mấy tiếng buổi tối quả thật rất tốn sức. Do đó, nếu một công cụ giúp tôi tóm tắt bài học để có thể tra lại sau khi học thì quả là tuyệt vời. Lúc này, tôi nghĩ ngay đến việc dùng AI để tóm tắt bài học, tra cứu video khi cần và cần biết chính xác thời điểm tương ứng với bài học để tôi có thể seek đến chính xác vị trí và xem chính xác nội dung cần tham khảo.


## Cần chuẩn bị những gì?
Để làm một "task" liên quan tới AI thì cần chuẩn bị 2 thứ:
 - 80% ở Data Engineering (chuẩn bị dữ liệu đầu vào).
 - 20% ở Prompt Engineering (viết prompt hợp lý).

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


## Prompt engineering

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


```python
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

[Bảo mật macOS MDM: Lỗ hổng enrollment dựa trên serial number]({{ site.baseurl }}/assets/2025/macOS_MDM.md)

## Kết luận

Qua bài viết vừa rồi, tôi hy vọng rằng các bạn có thể có cái nhìn sâu sắc hơn với cách viết prompt cho AI. AI rất thông minh, nhưng nếu dùng đúng prompt thì AI vừa thông minh vừa được việc. Chúc các bạn thành công./.
