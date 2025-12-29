# GIT

## 1. Basic Concept

<img width="1450" height="969" alt="image" src="https://github.com/user-attachments/assets/af3e5195-64fd-46fd-b96e-af6a46837030" />

Khi sử dụng Git ta có các vùng:
- Workspace (hay còn gọi là Working Directory): nơi chỉnh sửa code thực tế. File ở đây có thể:
  - Untracked: file mới chưa được git theo dõi. Hiển thị màu đỏ khi `git status`
  - Modified: là file đã được Git theo dõi (từng git add và commit trước đó) nhưng bị chỉnh sửa lại mà chưa git add lần nữa (tức thay đổi chưa Stage). Hiển thị màu đỏ khi `git status`
- Staging Area: chứa những file được đưa vào bằng lệnh `git add` và sẵn sàng commit
  - Chỉ những gì ở stage mới được commit
  - Hiển thị màu xanh lá khi `git status`
- Local repository: Lịch sử commit đã lưu vĩnh viễn trên local
- Remote repository: Là git repository

---
git diff hiển thị sự khác biệt (diff) giữa các trạng thái trong Git workflow.

- git diff: so sánh Working Directory với Staging Area. Dùng để xem thay đổi chưa git add (màu đỏ trong git status)
- git diff --staged hoặc git diff --cached → so sánh Staging Area với Last Commit. Dùng để xem những gì sắp commit (màu xanh trong git status)
- git diff HEAD	→ so sánh Working Directory với Last Commit. Dùng để xem tất cả thay đổi từ commit cuối
- git diff commit1 commit2	→ so sánh hai commit cụ thể

---
Trong Git, HEAD﻿ là con trỏ tham chiếu đến commit hiện tại bạn đang làm việc (commit cuối cùng trên nhánh đang checkout). HEAD﻿ trong Git là một con trỏ đặc biệt thể hiện "vị trí hiện tại bạn đang đứng" trong repository, tức là nó tham chiếu đến:

- Một nhánh (branch) mà bạn đang làm việc, ví dụ như main, develop...
- Một commit cụ thể trên nhánh đó — thường là commit mới nhất (cuối cùng) trên nhánh bạn đang checkout.

Nói cách khác, HEAD﻿ là sự kết hợp của nhánh và commit, đại diện cho commit cuối cùng hiện tại trên nhánh đang thao tác. 

Để hiểu rõ hơn về ý nghĩa của HEAD﻿ trong Git, bạn hãy tưởng tượng như thế này:

- Bạn có nhiều nhánh (branch) trong dự án, ví dụ như main, develop, feature1...

- Khi bạn làm việc, bạn chỉ làm việc trên một nhánh nhất định tại một thời điểm.

- HEAD﻿ chính là con trỏ giúp Git biết bạn đang đứng ở nhánh nào (ví dụ: main) và đang làm việc trên commit nào của nhánh đó.

- Khi bạn chuyển sang làm việc nhánh khác, Git sẽ cập nhật HEAD﻿ trỏ đến nhánh mới và commit cuối của nhánh đó.

- Nếu bạn checkout một commit cụ thể (không phải nhánh), Git sẽ để HEAD﻿ ở trạng thái "tách rời" (detached HEAD), tức nó trỏ thẳng đến commit đó thay vì nhánh nào, và bạn có thể làm việc trên phiên bản đó mà không ảnh hưởng đến nhánh.

Ví dụ đơn giản:


`git checkout main`

-> Lúc này HEAD trỏ tới commit cuối cùng của nhánh main

`git checkout develop`

-> HEAD trỏ sang commit cuối của nhánh develop

`git checkout abc1234`

-> HEAD tách rời, trỏ thẳng đến commit có ID abc1234


1. Cách xem HEAD hiện tại

`git rev-parse HEAD` -> in ra hash của commit mà HEAD đang trỏ tới.

`git log -1` -> in ra thông tin commit cuối cùng bạn đang đứng (HEAD).

2. Cách nhảy đến commit thứ 10 trên nhánh có 20 commit

`git log --reverse mybranch` -> Xem danh sách commit từ cũ đến mới của nhánh mybranch

`git log --reverse --pretty=format:"%h %s" mybranch` -> Xác định hash của commit thứ 10 trong danh sách này (có thể đếm thứ tự bằng cách đếm trên màn hình hoặc dùng lệnh)

`git checkout abc1234` -> chuyển sang commit đó 

---

#### Khi bạn đã `git commit` nhưng muốn bỏ commit, vẫn giữ thay đổi trong working tree để sửa/commit lại.

- Xóa 1 commit gần nhất: `git reset --soft HEAD~1` ⭢ Lịch sử mất commit cuối cùng, code vẫn còn và đang ở trạng thái staged.
​- Xóa nhiều commit (ví dụ 3 commit gần nhất): `git reset --soft HEAD~3`

Nếu muốn bỏ stage (chỉ giữ code trong working directory): `git reset`

#### Khi bạn đã `git commit` nhưng muốn xóa commit và xóa luôn code thay đổi (quay về trạng thái cũ)
- Xóa commit cuối cùng và vứt luôn thay đổi: `git reset --hard HEAD~1`
- Xóa toàn bộ commit local chưa push (vứt hết commit/thay đổi local, quay về đúng remote):
```
git fetch origin
git reset --hard origin/branch-name
```
​
---

### Lỗi `[rejected] master -> master (non-fast-forward)`
- Nguyên nhân là do nhánh master local của bạn đang đi sau nhánh master trên remote (remote master đã có thêm commit mới do ai đó push trước, hoặc bạn clone về lâu rồi không cập nhật), nên không được phép ghi đè lịch sử.
- Đây là cơ chế để bảo vệ lịch sử của remote do local master của bạn không chứa các commit mới
- Cách xử lý an toàn (nên dùng)
```
git pull --rebase origin master #lấy commit mới từ remote rồi đặt commit local của bạn lên trên lịch sử mới, giữ history đẹp.
# giải conflict (nếu có), rồi:
git push origin master
```
- Cách ép ghi đè remote (cẩn thận), chỉ dùng khi chắc chắn muốn xoá lịch sử trên remote và thay bằng lịch sử local:
```
git push --force origin master
```
Lưu ý cách này sẽ khiến các commit đang có trên remote mà bạn chưa pull bị mất, nên không dùng nếu còn người khác đang làm trên repo
