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
### Làm việc với Git repository
- Để kéo repository trên Git về local thì:
  - Dùng lệnh git clone nếu chưa có bản sao local. Git clone không yêu cầu git init trước khi clone Git repo về local do lệnh git clone tự động khởi tạo repository (tương đương git init) và tải toàn bộ code, lịch sử commit từ remote về, setup remote origin, và checkout branch mặc định (thường main hoặc master). Tránh git init trước clone vì sẽ tạo repo rỗng chồng chéo, gây lỗi khi clone vào thư mục đã init.
  - Dùng lệnh git pull nếu đã clone trước đó. 
- Dùng git init chỉ khi tạo repo mới hoàn toàn từ thư mục trống, chưa có remote. Sau git init, bạn phải tự git remote add origin <url> và git add/commit/push để kết nối remote.​

​

Kiểm tra thay đổi local: git status. Nếu có unstaged changes, commit (git add . && git commit -m "WIP") hoặc stash (git stash).
​

Kéo thay đổi mới: git pull origin main (thay main bằng branch của bạn, dùng --rebase nếu muốn lịch sử sạch: git pull --rebase origin main).
​

Push thay đổi local nếu cần: git push origin main.
​
---


git diff hiển thị sự khác biệt (diff) giữa các trạng thái trong Git workflow.

- git diff: so sánh Working Directory với Staging Area. Dùng để xem thay đổi chưa git add (màu đỏ trong git status)
- git diff --staged hoặc git diff --cached → so sánh Staging Area với Last Commit. Dùng để xem những gì sắp commit (màu xanh trong git status)
- git diff HEAD	→ so sánh Working Directory với Last Commit. Dùng để xem tất cả thay đổi từ commit cuối
- git diff commit1 commit2	→ so sánh hai commit cụ thể

---
Trong Git, HEAD﻿ là con trỏ tham chiếu đến commit hiện tại bạn đang làm việc (thường là commit mới nhất của branch đang checkout). HEAD là một khái niệm tồn tại trên cả local và remote. 
- HEAD local: HEAD trỏ vào branch hiện tại (ví dụ main, develop) và commit mới nhất hoặc trỏ vào branch hiện tại và một commit cụ thể nếu đang ở trạng thái detached HEAD. Có thể xem HEAD hiện tại bằng lệnh `git log --oneline --decorate -1`
  - Khi tạo commit mới: HEAD tự động cập nhật sang commit mới nhất của branch đó.
  - Khi checkout một commit cụ thể (không phải branch): HEAD trỏ trực tiếp vào commit đó → trạng thái detached HEAD.
​- HEAD remote: HEAD remote (origin/HEAD) là một con trỏ mềm trỏ đến branch mặc định của remote, tồn tại cả trên server và trên local repo, thường trỏ vào branch mặc định (ví dụ origin/main), cho biết branch nào được coi là “mặc định” khi clone. HEAD trên remote (thường là origin/HEAD) không trỏ trực tiếp vào một commit cụ thể như HEAD local, mà nó trỏ vào một branch mặc định của remote (ví dụ origin/main), và branch đó mới là thứ trỏ vào commit cụ thể. Mỗi branch remote (ví dụ origin/main, origin/develop) là một con trỏ trỏ vào commit cụ thể (commit SHA) trên remote. Khi có commit mới được push lên origin/main, con trỏ origin/main sẽ được cập nhật sang commit mới nhất, còn origin/HEAD vẫn chỉ trỏ đến origin/main (nếu main vẫn là branch mặc định). Bạn có thể xem HEAD của remote bằng lệnh `git remote show origin`. Ví dụ:
  - origin/HEAD → trỏ đến origin/main (branch mặc định).
  - origin/main → trỏ đến commit SHA cụ thể (ví dụ a1b2c3d...).
​​


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

Nếu đã git add + git commit nhưng chưa push lên origin, rồi chạy git pull từ remote về thì có thể bị lỗi, tùy vào tình huống cụ thể.
​
- Trường hợp không bị lỗi (fast-forward): Nếu remote branch chỉ có thêm commit ở phía trước (không sửa file nào mà local đã commit), Git sẽ tự động “fast-forward” khi chạy lệnh `git pull origin main`, Git chỉ cập nhật con trỏ nhánh local lên commit mới nhất của remote, không tạo merge commit, và commit local vẫn giữ nguyên, không bị lỗi.
​- Trường hợp có thể bị lỗi (cần merge/rebase): Nếu remote có commit sửa những file mà local đã commit (cùng file, cùng dòng), thì:
  - Với git pull (mặc định dùng merge): Git sẽ tự động tạo một merge commit để kết hợp lịch sử local và remote → Không bị lỗi, nhưng sẽ có thêm một commit merge trong lịch sử.
  - Với git pull --rebase origin main Git sẽ tạm lưu commit local chưa push, áp dụng commit mới từ remote và replay commit local lên trên → Lịch sử sạch hơn, nhưng nếu có conflict thì sẽ báo conflict và yêu cầu giải quyết.
​- Trường hợp bị lỗi nghiêm trọng: Git sẽ từ chối pull nếu:
  - Có unstaged changes (file đã sửa nhưng chưa git add)
  - Có uncommitted changes (file đã git add nhưng chưa git commit).
Lỗi thường thấy:
```
error: cannot pull with rebase: You have unstaged changes.
error: Please commit or stash them.
```

→ Phải commit/stash trước rồi mới git pull được.
​

Workflow an toàn là:
```
# 1. Commit/stash local changes trước
git add .
git commit -m "WIP: save local changes"

# 2. Pull với rebase để giữ lịch sử sạch
git pull --rebase origin main

# 3. Nếu có conflict: sửa, git add, git rebase --continue
# 4. Sau đó mới push
git push origin main
```

Nếu chỉ muốn lấy code mới mà không muốn merge/rebase ngay, có thể dùng:
```
git fetch origin    # chỉ tải code mới, không thay đổi nhánh hiện tại
git diff origin/main  # xem khác biệt trước khi merge/rebase
```
​
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


---

Git rebase là lệnh dùng để tái sắp xếp lịch sử commit, di chuyển các commit từ nhánh hiện tại lên đầu nhánh mục tiêu, tạo lịch sử tuyến tính sạch sẽ thay vì merge commit.
​

Cách hoạt động
Git tìm commit chung gần nhất giữa hai nhánh, tạm lưu commit của nhánh hiện tại, áp dụng commit từ nhánh mục tiêu, rồi replay commit tạm lưu lên trên. Kết quả: lịch sử commit thẳng hàng, không có merge commit thừa.
​

Lệnh cơ bản
Chuyển sang nhánh cần rebase: git checkout feature.
​

Rebase lên nhánh khác: git rebase main (hoặc git pull --rebase origin main như bạn từng dùng).
​

Nếu conflict: sửa file, git add, rồi git rebase --continue; hủy bằng git rebase --abort.
​

So với git merge
Merge: tạo commit mới kết hợp hai nhánh, giữ lịch sử phân nhánh (dễ theo dõi nhưng lộn xộn).
​

Rebase: lịch sử sạch, lý tưởng cho feature branch trước push shared branch, nhưng thay đổi commit hash (không rebase public branch).
​
---

Khác nhau giữa git branch và git branch -a


- git branch: Chỉ hiển thị các nhánh local (nhánh đang có trong repo trên máy bạn).
​- git branch -a (hoặc --all): Hiển thị tất cả nhánh, gồm nhánh local và các nhánh remote-tracking (ví dụ remotes/origin/main, remotes/origin/develop).


Nếu chỉ muốn chỉ xem nhánh remote-tracking: git branch -r.
​

Nếu bạn thấy nhánh remote nhưng chưa thấy local, có thể tạo local branch theo nó bằng `git switch -c <tên-local> --track origin/<tên-remote>` (hoặc `git checkout -t origin/<tên-remote>` tùy phiên bản Git).


Khi bạn chạy git push origin (không ghi rõ nhánh), Git sẽ cố đẩy nhánh local hiện tại (nhánh bạn đang checkout) lên remote tên origin đích mặc định là nhánh remote cùng tên với nhánh local (tức origin/<current_local_branch>) hoặc đẩy lên nhánh upstream đang link với nhánh local hiện tại (phụ thuộc vào cấu hình push.default là simple hay upstream). Nếu current local branch chưa link (chưa có upstream), Git thường sẽ báo lỗi kiểu “has no upstream branch” và gợi ý bạn dùng -u/--set-upstream để thiết lập.


Tác dụng của set upstream: ví dụ Nếu bạn set upstream của nhánh local staging thành origin/master (tức staging “track” origin/master), thì khi `git pull` khi đang ở staging sẽ mặc định pull/fetch+merge (hoặc rebase tùy config) từ origin/master về staging, vì upstream của staging là origin/master.
​

Khi dùng git push khuyến nghị nên viết rõ ràng nhánh local muốn đẩy lên `git push origin <branch-name>`, nghĩa là lấy nhánh local tên <branch-name> đẩy lên remote vào nhánh remote cũng tên <branch-name> (đích, mặc định là cùng tên) hoặc nhánh đã được set upstream với <branch-name>. Nếu local không có nhánh <branch-name>, lệnh sẽ thất bại kiểu “src refspec staging does not match any” vì không có nguồn để push.​

Nếu bạn muốn lấy nội dung từ main local và tạo/cập nhật nhánh staging trên remote, bạn cần chỉ rõ refspec nguồn:đích: `git push origin main:staging`

Lệnh để lần đầu push nhánh mới và muốn set upstream luôn `git push -u origin <branch-name>`. Từ lần sau bạn chỉ cần git push (hoặc git push origin) là được.
​

Lệnh để liệt kê các nhánh local và upstream tương ứng `git branch -vv` hoặc `git for-each-ref --format='%(refname:short) <- %(upstream:short)' refs/heads` (Xem upstream của mọi nhánh theo dạng “branch <- upstream”, cách này tiện khi bạn muốn nhìn quan hệ tracking của nhiều nhánh cùng lúc.)

Lệnh để xem upstream của nhánh hiện tại (1 dòng) `git rev-parse --abbrev-ref @{upstream}`

Lệnh để set upstream chứ không push `git branch -u origin/<remote-branch-name> <local-branch-name>`
​
Nếu bạn muốn có nhánh staging ở local trước rồi push theo tên nhánh:

```
git switch -c staging
git push -u origin staging
```
(Ý tưởng là tạo nhánh local staging, rồi push lên origin/staging và set upstream.)

---

`git clone -b <branch-name> <repo-url>` -> Lệnh này sẽ fetch các refs từ remote và checkout nhánh đó làm nhánh local hiện tại.
​
`git clone -b <branch-name> --single-branch <repo-url>` -> Chỉ clone đúng 1 nhánh​

`git checkout <branch-name>` -> checkout sang nhánh khác, hoạt động với cả local và remote 
