# SSM

Patch Manager là một tính năng của AWS Systems Manager dùng để tự động hóa quá trình vá lỗi (patching) cho các node được quản lý, bao gồm cả các bản cập nhật liên quan đến bảo mật và các bản cập nhật phần mềm khác. Tính năng này giúp quét các instance được quản lý để xác định những bản vá còn thiếu, sau đó quét báo cáo hoặc tự động cài đặt các bản vá phù hợp theo chính sách patching đã thiết lập.

Patch Manager hỗ trợ đa nền tảng hệ điều hành như nhiều bản phân phối Linux, macOS và Windows Server, cũng như các bản vá ứng dụng hạn chế (ví dụ với Windows). Bạn có thể thiết lập các chính sách patching tập trung cho nhiều tài khoản và khu vực AWS khác nhau, theo dõi trạng thái tuân thủ patch, và lập lịch thực thi patching theo khung giờ đã định.

Về version trong cảnh báo bạn thấy là version của configuration manager hoặc configuration script/template mà Quick Setup sử dụng để triển khai và quản lý chính sách Patch Manager trên các node. Phiên bản này đại diện cho bộ cấu hình, script, template (ví dụ CloudFormation stack hoặc SSM Document) mà AWS liên tục cập nhật để bổ sung tính năng mới, vá lỗi và cải thiện bảo mật cho việc quản lý patch tự động. Phiên bản mới (như v2.5.0 cảnh báo) sẽ đảm bảo các tính năng được cập nhật đầy đủ và vận hành ổn định hơn so với các phiên bản cũ.

Khi bạn sử dụng AWS Systems Manager Quick Setup, hệ thống sẽ tự động tạo hai hàm Lambda trong account của bạn có tên tương tự như: `delete-name-tags-...` `baseline-overrides-...`

Trong đó
- delete-name-tags-... là hàm Lambda được sử dụng để tự động xóa các thẻ (tags) tên không phù hợp hoặc thẻ dư thừa trên các tài nguyên (ví dụ như instance, managed nodes) do Quick Setup quản lý. Việc này giúp đảm bảo các thẻ tuân thủ theo chính sách và cấu hình của Quick Setup, tránh nhầm lẫn hoặc lỗi khi áp dụng cấu hình và patching.

- baseline-overrides-... là hàm Lambda liên quan đến việc quản lý các "patch baseline override" – tức là tùy chỉnh các chính sách vá lỗi (patching policies) khác với mặc định, được đặt theo dạng file cấu hình JSON trong S3 mà Lambda này sẽ lấy và áp dụng lên các node mục tiêu trong quá trình Patch Manager hoạt động. Nó giúp Quick Setup linh hoạt tùy biến chính sách patch theo từng môi trường hoặc nhóm instance riêng biệt.


Quick Setup cần tạo các hàm Lambda này như một phần của cấu hình tự động nhằm đảm bảo quản lý tags và patch baselines một cách chính xác, tự động hóa tối đa quá trình quản trị. Các Lambda này được gọi khi cần cập nhật hoặc kiểm tra trạng thái của node, nhằm giúp Quick Setup luôn giữ cho trạng thái hệ thống đúng theo cấu hình bạn đã thiết lập.
