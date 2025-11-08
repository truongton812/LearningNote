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
