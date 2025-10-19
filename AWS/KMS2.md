# Key Management Service (KMS)

## I. Định nghĩa và khái niệm
- Là dịch vụ của Amazon Web Services giúp tạo, lưu trữ, và quản lý các khóa mã hóa một cách an toàn với độ bền và bảo mật cực cao. S
- Các khóa mã hóa của KMS có thể dùng để mã hóa (encrypt), giải mã (decrypt) hoặc ký số (sign) dữ liệu được lưu trữ hoặc truyền qua các dịch vụ khác của AWS như S3, EBS, RDS hay Secrets Manager.​



- KMS key sử dụng hai cơ chế mã hóa:
  - Symmetric Keys (Khóa đối xứng)
  - Sử dụng một khóa duy nhất để thực hiện cả mã hóa và giải mã dữ liệu.
  - Loại khóa này dùng thuật toán AES-256, rất bảo mật và phổ biến.
  - Các dịch vụ AWS tích hợp với KMS đều sử dụng loại khóa đối xứng này để mã hóa dữ liệu (ví dụ: S3, EBS, RDS).
  - AWS không cho phép người dùng truy cập trực tiếp vào key này, bạn chỉ dùng API KMS để mã hóa/giải mã.
  - Symmetric key rất cần thiết cho mô hình Envelope Encryption: trong đó KMS dùng để mã hóa "key dữ liệu" trong khi dữ liệu lớn được mã hóa ngoài KMS.
  - Asymmetric Keys (Khóa bất đối xứng)
  - Bao gồm một cặp khóa: Public Key (khóa công khai) dùng để mã hóa hoặc xác thực, và Private Key (khóa riêng tư) dùng để giải mã hoặc ký số.
  - Các thuật toán hỗ trợ: RSA (2048/3072/4096-bit) và ECC (Elliptic Curve Cryptography).
  - Private Key được sinh và bảo vệ trong AWS KMS, không bao giờ bị lộ ra ngoài. AWS quản lý việc sử dụng private key qua API, bạn không thể tự lấy private key ra.
  - Public Key có thể tải xuống và sử dụng ngoài AWS (ví dụ cho các ứng dụng bên ngoài không gọi được API KMS).
  - Dùng cho các mục đích như encrypt/decrypt bên ngoài AWS hoặc thực hiện các thao tác ký và xác thực.
  - Nếu bạn cần mã hóa hoặc xác thực dữ liệu ngoài AWS, hoặc cho ứng dụng không gọi được KMS API, bạn dùng asymmetric key vì có thể cho người khác biết public key để mã hóa hoặc kiểm tra chữ ký.



- Khi tạo Customer Managed Key (CMK) trong AWS KMS có thể chỉ định thuộc tính key material origin, dùng để xác định nguồn gốc "key material" cho KMS key đó.​
- Key material là dãy số hoặc chuỗi bí mật (ví dụ 256 ký tự nếu là AES-256) mà tất cả các quá trình mã hóa, giải mã sẽ dùng. Key hiển thị trên AWS chỉ dùng để quản lý ( có các thuộc tính như tên key, ID, policy quản lý), nhưng phải có “key material” (bản thân dữ liệu khóa đó) thì AWS mới thực hiện được mã hóa/giải mã.
- AWS sẽ tự sinh key material giúp bạn khi tạo khóa, hoặc bạn tự upload/nhập vào nếu muốn kiểm soát tuyệt đối nguồn gốc của khóa đó.
- Một số tùy chọn phổ biến gồm:
  - AWS_KMS: AWS sẽ tự động sinh và quản lý key material (mặc định, khuyến nghị). Bạn không thấy hay thao tác với dữ liệu khóa gốc, tất cả được bảo vệ bởi phần cứng HSM đạt chuẩn FIPS.
  - EXTERNAL (Import key material): Bạn chủ động sinh khóa bí mật ngoài AWS (bằng thiết bị HSM hoặc phần mềm tuân thủ quy định riêng) và upload vào AWS KMS. Dùng khi có nhu cầu tự kiểm soát vòng đời hoặc tuân thủ quy định đặc biệt về kiểm soát/nguồn gốc khóa.
  - AWS_CLOUDHSM: Key material được sinh trong CloudHSM cluster do bạn quản lý trên AWS, phù hợp khi cần toàn quyền kiểm soát lifecycle và access audit.
  - EXTERNAL_KEY_STORE: Key material nằm hoàn toàn ngoài AWS (ở một external key manager/HSM do bạn sở hữu), dùng với dịch vụ Custom Key Store, đáp ứng các yêu cầu kiểm soát vật lý hoặc kỹ thuật nghiêm ngặt nhất.


---



Các loại key trong AWS KMS:
- AWS Managed Keys (khóa do AWS quản lý)
  - AWS tự động tạo và quản lý loại khóa này cho bạn khi bật mã hóa mặc định trên các dịch vụ (S3, EBS, RDS, v.v.).​
  - Bạn không thể truy cập trực tiếp khóa nhưng không cần lo quản lý vòng đời hay chính sách quyền truy cập.
  - Được đặt tên theo định danh dạng aws/service-name (ví dụ: aws/s3).
- Customer Managed Keys (CMK - khóa do người dùng tạo và quản lý)
  - Người dùng có thể tạo, cấu hình, gắn quyền truy cập, kích hoạt rotation, hoặc xóa khóa này.​
  - Có thể thiết lập automatic key rotation hằng năm.
  - Phù hợp cho các ứng dụng yêu cầu kiểm soát chi tiết hoặc tuân thủ quy định bảo mật.

