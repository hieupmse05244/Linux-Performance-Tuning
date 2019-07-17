- [1 - Hiểu về hệ điều hành Linux](#1)
    - [1.1 - Quản lý tiến trình Linux](#1.1)
        - [1.1.1 - Process (tiến trình) là gì?](#1.1.1)
        - [1.1.2 - Vòng đời (Life cycle) của một process](#1.1.2)
        - [1.1.3 - Thread (Luồng)](#1.1.3)
        - [1.1.4 - Ưu tiên tiến trình và nice level](#1.1.4)
        - [1.1.5 - Context switching (Chuyển đổi ngữ cảnh)](#1.1.5)
        - [1.1.6 - Xử lý ngắt](#1.1.6)
        - [1.1.7 - Trạng thái của process](#1.1.7)
        - [1.1.8 - Process memory segments](#1.1.8)
        - [1.1.9 - Linux CPU scheduler](#1.1.9)


---
---

<a name="1"></a>
# 1 - Hiểu về hệ điều hành Linux
<a name="1.1"></a>
## 1.1 - Quản lý tiến trình Linux

Việc quản lý process (tiến trình) là một trong những phần quan trọng nhất của bất cứ OS nào. Quản lý process hiệu quả cho phép một ứng dụng vận hành đều đặn (steadily) và hiệu quả (effectively)

Triển khai quản lý process của Linux cũng giống với của UNIX. Nó bao gồm việc lập lịch tiến trình (schedling), xử lý ngắt (interrupt), báo hiệu (signaling), ưu tiên tiến trình (prioritization), chuyển đổi tiến trình (switching), trạng thái tiến trình (state), bộ nhớ tiến trình (memory),...

Trong phần này, chúng ta sẽ thảo luận về những cơ bản của việc triển khai quản lý process trong Linux. Nó giúp bạn hiểu việc Linux kernel xử lý các process sẽ có ảnh hưởng như thế nào tới hiệu suất của hệ thống.
<a name="1.1.1"></a>
### 1.1.1 - Process (tiến trình) là gì?

Một process là một biểu diễn (instance) của quá trình thực thi (execution) chạy trên một processor (vi xử lý). Process sử dụng bất cứ tài nguyên (resources) nào mà nhân Linux (Linux kernel) có thể cung cấp (handle) để hoàn thành nhiệm vụ của nó.

Tất cả process chạy trên Linux OS đều được quản lý bởi cấu trúc **tast_struct**, nó được gọi là process descriptor (mô tả tiến trình). Một process descriptor chứa tất cả thông tin cần cho một single process (đơn tiến trình) có thể chạy, ví dụ như định danh (process id) các thuộc tính và tài nguyên xây dựng (construct) process. Hiểu cấu trúc của process giúp ta có thể hiểu điều gì là quan trọng cho việc thực thi  và hiệu suất của process.  Hình 1-2 cho thấy phác thảo của các cấu trúc liên quan đến thông tin process.

"Hình 1-2"
<a name='1.1.2'></a>
### 1.1.2 - Vòng đời của một tiến trình

Mọi process đều có vòng đời của riêng nó, bao gồm khởi tạo (creation), thực thi (execution), chấm dứt (termination) và gỡ bỏ (removal). Các giai đoạn (phase) này sẽ lặp lại hàng triệu lần (theo nghĩa đen) trong khi hệ thống up và running. Vì vậy, vòng đời của process rất qua trọng khi đứng từ quan điểm hiệu suất (performance perspective).

Hình 1-3 cho thấy vòng đời điển hình của các process

"Hình 1-3"

Khi một process tạo ra một process mới, process cha sẽ cung cấp một lời gọi hệ thống (system call) **fork()**. Khi lời gọi hệ thống **fork()** được cung cấp, nó lấy ra một mô tả tiến trình (process descriptor) cho tiến trình con và đặt một định danh (process id). Nó copy giá trị mô tả tiến trình của process cha cho mô tả tiến trính của process con. Vào lúc này, toàn bộ không gian địa chỉ của process cha chưa được copy, cả 2 process chia sẻ cùng một không gian địa chỉ.

Lời gọi hệ thống **exec()** sao chép chương trình mới tới không gian địa chỉ của process con. Vì cả 2 process chia sẻ cùng một không gian địa chỉ, việc viết dữ liệu chương trình mới (?) gây ra page fault exception (ngoại lệ lỗi trang). Vào lúc này, kernel sẽ gán một physical page (trang vật lý) mới cho process con.

Hoạt động hoãn lại này được gọi là *Copy On Write*. Process con thường thực thi chương trình của riêng chúng thay vì thực thi giống cha nó. Hoạt động này nhằm tránh những chi phí không cần thiết (unnecessary overhead) bởi việc sao chép toàn bộ không gian địa chỉ là một hoạt động rất chậm và không hiệu quả, nó sử dụng rất nhiều thời gian xử lý (processor time) và tài nguyên.

Khi việc thực thi chương trình đã hoàn thành, process con sẽ châm dứt (terminates) với một lời gọi hệ thống **exit()**. Lời gọi hệ thống **exit()** sẽ giải phóng (release) hầu hết cấu trúc dữ liệu (data structure) của process và thông báo cho process cha chấm dứt việc gửi tín hiệu. Vào lúc này, process sẽ được gọi là một *zombie process*  (tham khảo "[Zombie processes](#zombie_process)").

Process con sẽ không hoàn toàn bị xóa cho tới khi process cha biết việc chấm dứt của process con của nó bằng lời gọi hệ thống **wait()**. Ngay khi process cha được thông báo về việc chấm dứt của process con, nó sẽ xóa tất cả cấu trúc dữ liệu của process con và giải phóng process descriptor.

<a name='1.1.3'></a>
### 1.1.3 Luồng

Một thread (luồng) là một khối thực thi (execution unit) được tạo ra trong một single process. Nó chạy song song với các thread khác trong cùng một process. Chúng có thể chia sẻ tài nguyên như memory, không gian địa chỉ, các file mở, v.v.. Chúng có thể truy cập vào cùng một tập dữ liệu ứng dụng. Vì vậy, mỗi thread không nên thay đổi tài nguyên được chia sẻ cùng một thời điểm. Một thread còn được gọi là Light Weight Process (LWP). Việc thực hiện loại trừ lẫn nhau, khóa, tuần tự hóa, v.v., là trách nhiệm của ứng dụng người dùng (?).

Đứng trên quan điểm hiệu suất, việc tạo ra thread có chi phí thấp hơn việc tạo ra process vì một thread không cần sao chép tài nguyên khi tạo. Mặt khác, các process và thread có cùng các đặc điểm về mặt thuật toán lập lịch (scheduling algorithm). Kernel làm việc với cả hai theo cách tương tự.

"Hình 1-4"

Trong các bản thực thi của Linux hiện hành, một thread được hỗ trợ Giao diện hệ điều hành di động cho thư viện tương thích UNIX (Portable Operating System Interface for UNIX compliant library) - pthread. Một số triển khai thread có sẵn trong Linux. Sau đây là những bản được sử dụng rộng rãi:

- LinuxThreads

   LinuxThread có một số triển khai không tuân thủ với tiêu chuẩn POSIX. Native POSIX Thread Library (NPTL) đang thay thế LinuxThreads. LinuxThreads sẽ không được hỗ trợ trong bản phát hành Enterprise Linux trong tương lai.

- Native POSIX Thread Library (NPTL)

    NPTL ban đầu được phát triển bởi Red Hat. NPTL phù hợp hơn với các tiêu chuẩn POSIX. Bằng cách tận dụng các cải tiến trong kernel 2.6 như lời gọi hệ thống **clone()** mới, triển khai xử lý tín hiệu, v.v., nó có hiệu năng và khả năng mở rộng tốt hơn so với LinuxThreads.

    NPTL có một chút không tương thích với LinuxThreads. Một ứng dụng phụ thuộc vào LinuxThread có thể không hoạt động với việc triển khai NPTL.

- Next Generation POSIX Thread (NGPT)
    NGPT là phiên bản do IBM phát triển của POSIX thread library. Nó hiện đang được bảo trì và không có kế hoạch phát triển nào nữa.

Sử dụng biến môi trường **LD_ASSUME_KERNEL**, bạn có thể chọn thư viện Threads phù hợp với sử dụng.

<a name='1.1.4'></a>
### 1.1.4 - Ưu tiên tiến trình và nice level

*Process priority* (ưu tiên tiến trình) là một số dùng để xác định thứ tự xử lý của CPU, nó được xác định bởi độ ưu tiên động (dynamic priority) và độ ưu tiên tĩnh (static priority). Một process có độ ưu tiên cao hơn sẽ có nhiều cơ hội được chạy trên vi xử lý hơn.

Kernel tự động điều chỉnh mức độ ưu tiên động lên xuống khi cần bằng thuật toán heuristic dựa trên các hành vi và đặc điểm của process. Một user process có thể thay đổi mức độ ưu tiên tĩnh một cách gián tiếp thông qua việc sử dụng nice level (?) của process. Một process có mức ưu tiên tĩnh cao hơn sẽ có lát cắt thời gian (time slice) dài hơn (thời gian process có thể chạy trên bộ xử lý).

Linux hỗ trợ nice level từ 19 (mức ưu tiên thấp nhất) tới -20 (cao nhất). Giá trị mặc định là 0. Để đổi nice level của một chương trình thành số âm, cần quyền root (su)

<a name='1.1.5'></a>
### 1.1.5 - Chuyển đổi ngữ cảnh

Trong khi process thực thi, thông tin chạy của nó được lưu trữ trong các register (thanh ghi) trên processor (vi xử lý) và cache (bộ đệm) của nó. 
Tập dữ liệu được tải vào register cho quá trình thực thi process được gọi là *context* (bối cảnh). Để chuyển đổi các process, context của process đang chạy được lưu trữ và context của process chạy tiếp theo được khôi phục vào register. Process descriptor và khu vực - nơi được gọi là ngăn xếp kernel mode - đươc sử dụng để lưu trữ context. Quá trình này được gọi là *context switching*.

Có quá nhiều context switching là điều không mong muốn vì processor phải xóa register và cache của nó mỗi lần để nhường chỗ cho process mới. Nó có thể gây ra vấn đề hiệu suất! 

Hình 1-5 minh họa cách context switching hoạt động

"Hình 1-5"

<a name='1.1.6'></a>
### 1.1.6 - Xử lý ngắt

Xử lý ngắt là một trong những nhiệm vụ ưu tiên cao nhất. Ngắt thường được tạo bởi các thiết bị I/O như card mạng, bàn phím, bộ điều khiển đĩa, bộ điều hợp nối tiếp, v.v. 

Trình xử lý ngắt thông báo cho nhân Linux một sự kiện (chẳng hạn như đầu vào bàn phím, ethernet frame arrival(?), v.v.). Nó báo cho kernel ngắt quá trình thực thi process và thực hiện xử lý ngắt càng nhanh càng tốt vì một số thiết bị yêu cầu phản hồi nhanh. Điều này rất quan trọng cho sự ổn định của hệ thống. 

Khi tín hiệu ngắt đến kernel, kernel phải chuyển process thực thi hiện tại sang một process mới để xử lý ngắt. Nó có nghĩa là các ngắt gây ra context switching và do đó, một số lượng đáng kể các ngắt có thể làm giảm hiệu suất.

Trong triển khai Linux, có hai loại ngắt:

- *Ngắt cứng (hard interrupt)*: được tạo cho các thiết bị yêu cầu khả năng đáp ứng (ngắt I / O đĩa, ngắt bộ điều hợp mạng, ngắt bàn phím, ngắt chuột).

- *Ngắt mềm (soft interrupt)*: được sử dụng cho các tác vụ xử lý có thể được hoãn lại (hoạt động TCP / IP, hoạt động giao thức SCSI, v.v.).

Bạn có thể xem thông tin liên quan đến ngắt cứng tại `/proc/interrupts`

Trong môi trường multi-processor (đa vi xử lý), các ngắt được xử lý bởi mỗi processor. Liên kết ngắt với một single physical processor (vi xử lý vật lý đơn) có thể cải thiện hiệu suất hệ thống. Để biết thêm chi tiết, tham khảo "[4.4.2 - CPU affinity for interrupt handling](#4.4.2)"

<a name='1.1.7'></a>
### 1.1.7 - Trạng thái của process

Mỗi process có trạng thái riêng của nó, nó show ra chuyện gì đang xảy ra hiện tại bên trong process. Trạng thái của process thay đổi trong khi process thực thi. Một số trạng thái khả thi (possible states) (?):
- TASK_RUNNING: 

    Trong trạng thái này, một process đang chạy (running) trên một CPU hoặc đang đợi (waiting) để chạy trong queue (?) - run queue.

- TASK_STOPPED: 

    Ở trạng thái này, một process bị gián đoạn (suspended) bởi một tín hiệu nhất định (ví dụ như **SIGINT**, **SIGSTOP**). Process đó đang đợi để được tiếp tục bởi một tín hiệu như **SIGCONT**.

- TASK_INTERRUPTIBLE: 

    Ở trạng thái này, process bị gián đoạn và đợi cho đến khi một điều kiện nhất định được đáp ứng. Nếu process trong trạng thái này và nó nhận một tín hiệu stop, trạng thái của nó sẽ bị thay đổi và hoạt động sẽ bị ngắt. 
    
    Một ví dụ điển hình của một TASK_INTERRUPTIBLE process là một process đang đợi cho một ngắt bàn phím.

- TASK_ZOMBIE: 

    Sau khi một process bị chấm dứt bởi lời gọi hệ thống **exit()**, process cha của nó không biết về việc chấm dứt này. Ở trạng thái này, nó đợi cho tới khi process cha được thông báo và sau đó giải phóng tất cả cấu trúc dữ liệu.

"Hình 1-6"

<a name="zombie_process"></a>
#### Zombie process

Khi một process đã sẵn sàng chấm dứt và nhận được một tín hiệu để làm vậy, nó thường mất một lúc để kết thúc tất cả các task (ví dụ như đóng các file đang mở) trước khi tự kết thúc. 

Trong khoảng thời gian ngắn đó, nó là một zombie process.

Sau khi process đã hoàn thành tất cả các task tắt máy này, nó báo cáo tới process cha rằng nó sắp chấm dứt. Thỉnh thoảng, một zombie process không thể tự chấm dứt, trong trường hợp đó, nó hiển thị trạng thái Z (zombie).

Không thể dùng lệnh kill lên zombie process vì nó được coi là đã chấm dứt. Nếu bạn không thể thoát khỏi một zombie process, bạn có thể kill process cha của nó và sau đó zombie process sẽ biến mất.

Tuy nhiên, nếu process cha của nó là một init process (tiến trình khởi tạo), bạn không nên kill nó. Init process là một process rất quan trọng, vì vậy có thể cần phải khởi động lại để thoát khỏi zombie process

<a name='1.1.8'></a>
### 1.1.8 - Process memory segments

<a name='1.1.9'></a>
### 1.1.9 - Linux CPU scheduler