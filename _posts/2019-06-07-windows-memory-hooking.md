---
layout: post
title: "An In-depth Look: Windows Memory Hooking"
date: 2019-06-07 00:00:00.000000000 +07:00
type: post
published: true
status: publish
categories:
- Tutorials
tags:
- Hooking
excerpt: "Hooking không phải là một khái niệm mới khi giải quyết các vấn đề trong lập trình. Tuy nhiên, nếu nhìn nó theo cách nghiêm túc hơn, hẳn rằng bạn cũng sẽ có những trải nghiệm thú vị giống như tôi. Nếu bạn thích bài viết này, hãy tài trợ."
---

## Kêu gọi tài trợ
Xin chào tất cả các bạn độc giả thân mến!

Tôi viết bài viết này với mong muốn có thể đóng góp một chút công sức của mình cho cộng đồng an toàn thông tin Việt Nam. Tôi cũng mong muốn các bạn có thể đóng góp cho cộng đồng giống như tôi, nhưng bằng vật chất: Quỹ [Cơm có thịt](https://www.facebook.com/comcothit/) [http://tnvc.vn/](http://tnvc.vn/) đang cần sự giúp đỡ của cộng đồng hơn ai hết. Số tiền ủng hộ từ bài viết trước đã đủ cho develbranch.com duy trì mọi thứ trong năm. Quỹ cơm có thịt cần nhiều hơn develbranch. Đúng như tên gọi của nó, _Cơm có thịt_ đem đến niềm vui, hạnh phúc với các em nhỏ vùng cao ngoan hiền, đang sống ở những nơi nghèo khó, bằng những đóng góp nho nhỏ - ít thôi nhưng đều đặn. Tôi rất thích câu nói này của ông Trần Đăng Tuấn: "Nếu bạn đồng hành cùng Cơm Có Thịt, thì đó chỉ là do mệnh lệnh từ trái tim của bạn". Nếu có thể được, tôi mong các bạn hãy ủng hộ trực tiếp cho Quỹ (đừng trung gian qua tôi).

![Cơm có thịt]({{ site.baseurl }}/assets/2019/06/com_co_thit.jpg)

## Giới thiệu
**Hooking** có lẽ không còn xa lạ với nhiều người làm trong lĩnh vực phần mềm nói chung và an toàn thông tin nói riêng. Hiểu theo cách đơn giản: Hooking là một phương pháp, hoặc 1 cách làm thay đổi luồng hoạt động của chương trình. Mục đích của việc thay đổi này có thể để ghi log, đánh dấu luồng hoạt động hoặc kiểm soát input cũng như output của một hàm, một đoạn mã hoặc một lệnh bất kì trong chương trình.

Trong bài viết này, tôi sẽ viết về kĩ thuật hooking trên nền tảng Microsoft Windows.

**_Ví dụ_**: Chúng ta có một hàm đơn giản sau: Hàm này làm nhiệm vụ cộng 2 số.

```
int add(int a, int b) {
	return a + b;
}
```

Khi biên dịch ra mã máy, mã chương trình có thể giống như sau (các mã khác nhau phụ thuộc nhiều vào trình biên dịch). Trong ví dụ này, tôi giả định tôi biên dịch chương trình bằng trình biên dịch 32bit:

Nếu biên dịch tối ưu, có thể viết như sau:

```
mov  eax, dword ptr [esp + 4] ;  first parameter
add  eax, dword ptr [esp + 8] ;  second parameter
ret
```

Hoặc tường minh hơn, chúng ta viết:

```
0| mov  edi, edi
1| push ebp 
2| mov  ebp, esp
3| mov  eax, dword ptr [ebp +  8] ;  first parameter
4| add  eax, dword ptr [ebp + 12] ;  second parameter
5| pop  ebp
6| ret
```

**_Tình huống 1_**: Chúng ta muốn kiểm tra hoặc thay đổi dữ liệu input, chúng ta cần đặt 1 lệnh kiểm tra ở dòng 2.

**_Tình huống 2_**: Chúng ta muốn kiểm tra hoặc thay đổi dữ liệu output, chúng ta cần đặt 1 lệnh kiểm tra ở dòng 5

Phương pháp hook thường thấy nhất là chúng ta sẽ thiết lập 1 lệnh nhảy không điều kiện (JMP) tới đoạn code chứa các lệnh kiểm tra. Sau khi thực hiện xong đoạn code mới này, chúng ta sẽ thực hiện nhảy không điều kiện 1 lần nữa về đoạn code ban đầu.

**_Tình huống 1_**: Chúng ta muốn kiểm tra hoặc thay đổi dữ liệu input:
Đoạn code trên cần bị thay đổi thành đoạn code dưới đây

```
0| jmp  CHECKING_PARAMETER
3| ADD_NUMBERS:
3| mov  eax, dword ptr [ebp +  8] ;  first parameter
4| add  eax, dword ptr [ebp + 12] ;  second parameter
5| pop  ebp
6| ret
7|
8| CHECKING_PARAMETER:
   ;; kiểm tra nội dung 2 tham số ở đây
   ;  Thực thi các lệnh ban đầu, giống như chưa bị Hook
   mov  edi, edi
   push ebp 
   mov  ebp, esp
9| jmp ADD_NUMBERS

```


**_Tình huống 2_**: Chúng ta muốn kiểm tra hoặc thay đổi dữ liệu output:
```
 0| mov  edi, edi
 1| push ebp 
 2| mov  ebp, esp
 3| mov  eax, dword ptr [ebp +  8] ;  first parameter
 4| add  eax, dword ptr [ebp + 12] ;  second parameter
 5| pop  ebp
 6| JMP  CHECK_RESULT 
 7|
 8|
 9| CHECK_RESULT:
10| ;; Kiểm tra dữ kiệu trả về ( Nội dung thanh ghi eax )
    ;; Thực thi các lệnh ban đầu, giống như chưa bị Hook
11|	ret
```


Rõ ràng, trong cả hai tình huống trên, chúng ta cần thay đổi đáng kể đoạn code ban đầu, để có thể thêm các lệnh nhảy không điều kiện nhằm chuyển hướng thực thi và làm nhiệm vụ mà chúng ta cần. Câu hỏi đặt ra: 
 - Nếu đoạn code quá nhỏ, không đủ cho một lệnh JMP thì sao?
 - Trong trường hợp đoạn code bị kiểm tra để phát hiện sự thay đổi thì sao? Ví dụ đoạn code bị check CRC liên tục?
 
Để giải quyết hai vấn đề này, tôi sẽ trình bày hai phương pháp dưới đây.

## Breakpoint hooking

Đối với các đoạn code nhỏ, ví dụ các block code chỉ có 1 tới 4 bytes, chúng ta không có đủ chỗ cho một lệnh nhảy. Một lệnh JMP không điều kiện cần 5 bytes. Chúng ta có thể nghĩ tới phương pháp Breakpoint hooking. Cách thức thực hiện như sau:
 - Inject một DLL vào không gian bộ nhớ của tiến trình.
 - Cài đặt một Vector xử lý Exception (Exception handler)
 - Chuyển hướng thực thi bên trong Exception handler
 - Inject ngắt `int3` (opcode là 0xCC, độ dài chỉ 1 byte) vào vị trí cần hook. Hoặc dùng các hàm sau để inject code: `OpenProcess`, `VirtualProtectEx`, `WriteProcessMemory`. Đây là các thao tác đơn giản, tôi xin dành cho bạn đọc.
 
Trong quá trình thực thi, khi thực thi đến lệnh `int 3`, hệ thống sẽ phát sinh ra exception. Mã của exception này là `STATUS_BREAKPOINT`. Bên trong hàm handler, chúng ta cần kiểm tra giá trị của thanh ghi EIP/RIP. Nếu giá trị này là hàm đang cần hook, chúng ta gán lại giá trị mới cho thanh ghi EIP/RIP. Sau khi thực thi xong, chúng ta cần khôi phục lại ngữ cảnh cũ: Nội dung các thanh ghi và thực thi lệnh cũ. Nên nhớ rằng trước khi patch int3, chúng ta phải lưu lại toàn bộ instruction cũ, để sau này còn dùng lại khi thực thi chương trình.
 
Hàm xử lý:
```
LONG CALLBACK HookExceptionFilter(__in PEXCEPTION_POINTERS pExceptionInfo)
	{
		if (pExceptionInfo->ExceptionRecord->ExceptionCode == STATUS_BREAKPOINT) // This is going to return true whenever any of our int3 is executed.
		{
			DWORD dwNewLocation = GetNewLocation(pExceptionInfo->ContextRecord->Eip); // EIP contains the current location
			if (dwNewLocation != (DWORD_PTR)-1) // Here we check to see if the instruction pointer is at the place where we want to hook.
			{
				pExceptionInfo->ContextRecord->Eip = dwNewLocation;
				return EXCEPTION_CONTINUE_EXECUTION;
			}
		}
		return EXCEPTION_CONTINUE_SEARCH;
	}
```

Cài đặt handler:
```
PVOID g_ExceptionHandle = NULL;
g_ExceptionHandle = AddVectoredExceptionHandler(1, HookExceptionFilter);
```

Remove Handler:
```
RemoveVectoredExceptionHandler(g_ExceptionHandle);
```

## PageGuard Hooking

Đối với các đoạn code bị check CRC liên tục, chúng ta sẽ không được phép thay đổi một byte nào trong đoạn code gốc. Phương pháp này thực hiện như sau:
 - Inject một DLL vào không gian bộ nhớ của tiến trình.
 - Cài đặt một Vector xử lý Exception (Exception handler).
 - Đổi thuộc tính của page chứa vùng code cần hook thành `PAGE_GUARD` hoặc `PAGE_NOACCESS`. Dùng VirtualProtect
 - Chuyển hướng thực thi bên trong Exception handler

Bất khi nào code được thực thi trong vùng nhớ bị đánh dấu là `PAGE_GUARD`, hệ thống sẽ sinh ra một exception. Bên trong hàm kiểm tra Exception handler, chúng ta cần kiểm tra địa chỉ đang thực thi. Nếu địa chỉ đó là địa chỉ hàm cần hook, chúng ta chuyển hướng nó tới hàm đích.

Cần lưu ý: 

1. Thuộc tính `PAGE_GUARD` sẽ tự động bị xóa khi phát sinh exception. Do đó để tiếp tục hook, chúng ta cần thiết lập lại thuộc tính này cho vùng nhớ đó. 
2. Có thể dùng thuộc tính `PAGE_NOACCESS`. Hệ thống sẽ sinh `STATUS_ACCESS_VIOLATION` khi chúng ta thực thi đoạn code.
3. `STATUS_SINGLE_STEP` Đây không hẳn là một lỗi, để có thể coi là một exception. `STATUS_SINGLE_STEP` được sinh ra khi một lệnh được thực thi (giống như đang bị debug).


**_Tại sao chúng ta cần STATUS_SINGLE_STEP?_**

Một hàm sẽ nằm trong 1 page, nhưng điểm bắt đầu của hàm thì chưa hẳn là nằm ở đầu page. Trong trường hợp một đoạn code - không phải đoạn code cần hook - nhưng nằm cùng page với đoạn code cần hook thực thi, nó cũng sinh ra exception. Rõ ràng đây không phải là điều chúng ta mong muốn. Chúng ta cần là thiết lập lại thuộc tính `PAGE_GUARD` hoặc `PAGE_NOACCESS` như trước. Vấn đề lại tiếp tục xảy ra, đoạn code sinh exception liên tục nhưng không thực thi. Để giải quyết, chúng ta cần bật cờ `STATUS_SINGLE_STEP`. Lúc này, chương trình sẽ tiếp tục thực thi nhưng sẽ liên tục gọi vào hàm handler của chúng ta với mã của exception là `STATUS_SINGLE_STEP`. Chúng ta sẽ kiểm tra giá trị của thanh ghi `EIP`, nếu đúng hàm cần hook thì sẽ thiết lập thuộc tính `PAGE_GUARD` hoặc `PAGE_NOACCESS` như trước.


Code và hình vẽ tôi tham khảo ở [https://guidedhacking.com/threads/veh-hooking-aka-pageguard-hooking-an-in-depth-look.7164/](https://guidedhacking.com/threads/veh-hooking-aka-pageguard-hooking-an-in-depth-look.7164/)


![Memory page]({{ site.baseurl }}/assets/2019/06/veh_hooking_page.png)


```
DWORD dwOld;
VirtualProtect((void*)0x08048fb7, 1, PAGE_EXECUTE | PAGE_GUARD, &dwOld); // This sets the protection for whatever memory page that 0x08048fb7 is located in to PAGE_EXECUTE & PAGE_GUARD.
                                                                          // Which is going to cause an exception for any address accessed in that memory page, including the one we're after.

AddVectoredExceptionHandler(1, HookExceptionFilter); // Registers our vectored exception handler which is going to catch the exceptions thrown.
 
LONG CALLBACK HookExceptionFilter(__in PEXCEPTION_POINTERS pExceptionInfo)
{
    if (pExceptionInfo->ExceptionRecord->ExceptionCode == STATUS_GUARD_PAGE_VIOLATION) // This is going to return true whenever any of our PAGE_GUARD'ed memory page is accessed.
    {
        if (pExceptionInfo->ContextRecord->Eip == 0x08048fb7) // Here we check to see if the instruction pointer is at the place where we want to hook.
        {
            dwJmpBack = (DWORD*)(pExceptionInfo->ContextRecord->Esp + 0); // Find the return address for the JMP/EIP back into the target program's code.
            dwJmpBack = (DWORD)pExceptionInfo->ContextRecord->Eip + 5; // or just skip X number of bytes.
            pExceptionInfo->ContextRecord->Eip = (DWORD)hkFunction; // Point EIP to hook handle.
        }
     
        pExceptionInfo->ContextRecord->EFlags |= 0x100; //Set single step flag, causing only one line of code to be executed and then throwing the STATUS_SINGLE_STEP exception.
     
        return EXCEPTION_CONTINUE_EXECUTION; // When we return to the page, it will no longer be PAGE_GUARD'ed, so we rely on single stepping to re-apply it. (If we re-applied it here, we'd never move forward.)
    }
     
    if (pExceptionInfo->ExceptionRecord->ExceptionCode == STATUS_SINGLE_STEP) // This is now going to return true on the next line of execution within our page, where we re-apply PAGE_GUARD and repeat.
    {
        DWORD dwOld;
        VirtualProtect((void*)0x08048fb7, 1, PAGE_EXECUTE | PAGE_GUARD, &dwOld);
     
        return EXCEPTION_CONTINUE_EXECUTION;
    }
     
    return EXCEPTION_CONTINUE_SEARCH;
}
```

## Kết luận
Qua bài viết vừa rồi, tôi hi vọng rằng các bạn có thể thực hiện hook một số hàm đặc biệt các hàm có kích thước nhỏ, hoặc không thể hook tường minh:

- Xác định vị trí cần hook 
- Cài đặt exception handle và đổi thanh ghi EIP/RIP thành địa chỉ mới.
- Inject ngắt 3 hoặc thay đổi thuộc tính của page thành `PAGE_GUARD`
- Xử lý chuyển hướng bên trong hàm handler

