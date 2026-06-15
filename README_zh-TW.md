# Dunnet-like Text Adventure Game

> [English](README.md) | [繁體中文]

主程式檔案：`dunnet`

使用語言：C Shell (`csh` / `tcsh`)

作業系統：Ubuntu 20.04.6 LTS

Linux kernel version：5.15.0-139-generic

## 壹、專案簡介

本專案為課程UNIX system programing的作業，作業要求完成一個類似 **Dunnet 的**文字冒險遊戲，所有操作與輸出結果均需與原始 Dunnet 一致，不過不需要完成完整的遊戲流程，僅需實做到能讓玩家透過 pokey 找出森林中隱藏的寶藏即可。

此專案使用 `csh` script 來進行撰寫，玩家可以在不同房間之間移動、檢查物品、撿取或放下道具，並透過與場景中的物件互動推進遊戲。遊戲中也包含一個模擬 UNIX 環境的電腦 - pokey。玩家啟動 pokey 並成功登入後，可以輸入簡化版 UNIX 指令，例如 `ls`、`cd`、`pwd`、`cat` 與 `uncompress`……等等。

---

## 貳、執行方式

### 執行環境

Linux 或 UNIX 系統，且需支援：

- `csh` 或 `tcsh`
- `tar`
- `gunzip`
- 一般 UNIX 工具，例如：`sed`、`tr`、`grep`、`ls`、`pwd`、`mv`、`touch` 與 `chmod`

### 執行步驟

將 dunnet 與 `filesystem.tar` 放在同一個資料夾中。

接著在 ternumal 輸入：

```bash
chmod +x dunnet
./dunnet
```

即可啟動遊戲。

### 測試步驟

請先確保當前Linux或UNIX已經下載`emacs`：

```bash
sudo apt update && sudo apt install -y emacs
```

接著將 dunnet 與 filesystem.tar 同一個資料夾中。

開啟terminal輸入：

```bash
./dunnet
```

即可開始遊戲。

#### 檢查自製dunnet輸出結果

若是想知道自製 dunnet 與原始 dunnet 是否一致，可以按照以下步驟執行，或參考 demo files 資料中的 ReadMe.md。

首先將 dunnet、filesystem.tar 與三份測試檔案放入同一個資料夾中。

開啟terminal，輸入：

```bash
./dunnet < testactions > my_output.txt
```

這會將測試結果輸出到 my_output.txt 當中，testactions可更換成其他的測試檔案名稱。

然後複製所有測試檔案到另外一個資料夾。

開啟terminal，輸入：

```bash
emacs -batch -l dunnet < testactions > standard_output.txt
```

這會將原始dunnet的執行結果出到 standard_output.txt 當中。

接著我們 my_output.txt 和 standard_output.txt 移到同一個資料夾，開啟 terminal 輸入：

```bash
diff -y -w my_output.txt standard_output.txt
```

即可查看自製 dunnet 執行結果與原始 dunnet 執行結果的差異。

### 程式初始化流程說明

遊戲啟動時，程式會先將目前工作目錄切換至 script 所在的位置，再尋找 `filesystem.tar`。

程式會檢查：

```bash
./filesystem.tar
```

如果找到壓縮檔，程式會刪除舊的 `filesystem` 資料夾，重新解壓縮乾淨的檔案系統。這樣每次啟動遊戲時，物品位置、權限與房間狀態都會重置。

如果找不到 `filesystem.tar`，但目前已有 `filesystem` 資料夾，程式仍可繼續執行，但可能會導致遊戲無法正常進行，例如：key 被放在房子中，使得玩家無法進入房子。

---

## 參、整體實作方式

### 1. 使用資料夾表示房間

每個房間都是 `filesystem/pokey/rooms/` 底下的一個資料夾，所有房間如下：

```plain
bear-hangout
building-front
computer-room
dead-end
e-w-dirt-road
fork
hidden-area
mailroom
ne-sw-road
old-building-hallway
se-nw-road
```

每個房間資料夾中都包含 description， 其中儲存該房間的文字說明。

需要注意的是，在原本的 Dunnet 中，**使用 pokey 無法找到 mailroom 資料夾的存在**！但在此自製的 Dunnet 中可以找到。

```notion

                                         None
                                           A
                                           |n         
                                 w         |            e
                  computer-room <- old-building-hallway -> mailroom
                                          /
                                         /ne(s in reverse)
                                        /
                                 building-front
                                      /
                                     /ne
                                    /
                               ne-sw-road
                                  /
                                 /ne
         e                e     /
dead-end -> e-w-dirt-road -> fork
                                \
                                 \se
                                  \
                               se-nw-road
                                   \
                                    \se
                                     \
                                 bear-hangout
                                     /
                                    /sw
                                   /
                              hidden-area
```

### 2. 使用 symbolic link 表示移動方向

若一個房間可向東 (e) 移動，則在資料夾中會有對應的 symbolic link — e 存在，該 link 連接到下一個房間的資料夾。

當玩家輸入方向時，程式會先檢查對應的 symbolic link 是否存在，若存在則透過 `cd` 指令進入 link 連結的資料夾中。

接著，程式再使用`pwd -P`取得 symbolic link 解析後的實體路徑，用於判斷玩家目前所在的房間。

所有移動指令如下：

```bash
u
up

d
down

n
north

s
south

e
east

w
west

ne
northeast

nw
northwest

se
southeast

sw
southwest
```

### 3. 使用 .o 檔案表示物品

可以互動的物品主要以 `.o` 檔案表示，例如：

```notion
shovel.o
lamp.o
food.o
key.o
board.o
bracelet.o
paper.o
```

不過部分 `.o` 檔案不可撿取，像是：

```notion
bear.o
boulder.o
```

玩家撿取物品時，程式會將對應檔案移動到：

```bash
filesystem/pokey/usr/toukmond
```

此資料夾同時代表玩家的 inventory，以及登入 pokey 後的 home directory。

### 4. 使用 symbolic link 處理物品別名

部分物品具有多種名稱。例如：

```plain
bracelet
emerald
```

都代表：

```plain
bracelet.o
```

```plain
board
card
chip
cpu
```

都代表：

```plain
board.o
```

程式會統一將它們對應至 `.o` 檔案的名稱，以避免同一個物品被當成多個獨立物品。

### 5. 使用時間戳記控制輸出順序

玩家撿取或放下物品時，程式會執行：

```bash
touch "$object"
```

來更新物品的 timestamp。

之後使用：

```bash
ls -tr
```

依照檔案更新時間選取檔案，使物品在輸入`get all`、`i`、`l`指令時可以依照互動順序顯示。

---

## 肆、指令輸入

玩家輸入指令後，程式會先讀取完整輸入：

```bash
set input = $<:q
```

接著暫時停用 filename substitution：

```bash
set noglob
```

這可以避免玩家輸入：

```bash
x *
```

時，`*` 被外層 shell 展開為目前資料夾中的所有檔案。

### 大小寫處理

一般遊戲模式的輸入不區分大小寫：

```bash
set input = `echo "$input" | tr 'A-Z' 'a-z'`
```

例如：

```bash
GET SHOVEL
Get Shovel
get shovel
```

都會被視為：

```bash
get shovel
```

### 分隔符號處理

在 Dunnet 的一般冒險模式中，以下字元都視為 delimiters：

```bash
空格
:
;
,
```

程式會將連續 delimiters 統一轉換為一個空格：

```bash
set input = `echo "$input" | sed 's/[ :;,][ :;,]*/ /g'`
```

因此，下列輸入會具有相同效果：

```bash
x shovel
x:shovel
x;shovel
x,shovel
x,:;;shovel
```

### 參數陣列與 padding

正規化後，程式將輸入拆成參數陣列：

```bash
set args = ( $input:x )
```

為了避免存取不存在的元素時出現 `error: Subscript out of range`

程式會補上數個空字串：

```bash
set args_padded = ( $args:q "" "" "" "" )
```

例如，玩家只輸入：

```bash
get
```

程式仍可安全存取：

```bash
$args_padded[2]
$args_padded[3]
$args_padded[4]
```

---

## 伍、一般遊戲模式指令

| 指令 | 作用 | 實作方式 |
| --- | --- | --- |
| `n`, `s`, `e`, `w`, </br> `ne`, `nw`, `se`, `sw`, </br> `u`, `d` | 移動到其他房間 | 檢查目前資料夾中是否存在相同名稱的 symbolic link，存在時使用 `cd` 進行移動，並透過 `pwd -P` 取得該房間絕對路徑。 |
| `l`, </br> `look`, </br> `x`, </br> `examine`, </br> `read` | 用於查看目前房間詳細資訊 | 讀取目前房間中的 `description`，再讀取房間內的所有 `.o` 檔案，將檔名轉換為對應的物品描述並輸出。 |
| `l <item>`, </br> `look`&nbsp;`<item>`, </br> `x`&nbsp;`item`, </br> `examine`&nbsp;`<item>`, </br> `read`&nbsp;`<item>` | 檢查指定物品；若無輸入物品名稱，則變成輸出當前房間資訊，效果與`l`相同 | 根據玩家輸入的名稱，使用 `cat` 輸出對應物品的 `.o` 檔案 *(或一般檔案)* 中的文字內容。 |
| `i`, </br> `inventory` | 查看玩家所持有的物品 | 列出 `../../usr/toukmond/`中，所有 `.o` 檔案的名稱，忽略 symbolic link。 |
| `get <item>`, </br> `take <item>` | 撿取指定物品 | 使用 `mv` 將目前房間內指定的可撿取物品移動到 `../../usr/toukmond/`，再用 `touch` 更新 timestamp。 |
| `get all`, </br> `take all` | 撿取房間內所有可撿取物品 | 使用 `ls -tr *.o` 找出物品，跳過隱藏物品 *(以`.`作為檔名開頭)* 、不可撿取物品，再依序移動到 inventory — `../../usr/toukmond/`。    若有撿取到帶有 symbolic link 的物品 (例如: cpu、bracelet)，需要同步移動 symbolic link 到 inventory 中。 |
| `drop <item>`, </br> `throw <item>` | 放下物品 | 使用 `mv` 將 inventory 內的物品移動到目前房間，再用 `touch` 更新 timestamp。 |
| `dig` | 挖掘地面 | 先檢查 inventory 中是否擁有`shovel.o`。若玩家位於指定房間 `fork`，會顯示隱藏的 CPU board。 |
| `put`&nbsp;`<item1>`&nbsp;`in`&nbsp;`<item2>`, </br> `insert`&nbsp;`<item1>`&nbsp;`in`&nbsp;`<item2>` | 將物品放入另一個物品 | 目前主要用於將 CPU board 放入 computer，使 computer 啟動。  介係詞沒有硬性規定要使用`in`。 |
| `type` | 操作 computer | 只有位於 `computer-room` 時有效。若 computer 已啟動，玩家可以進入 `pokey` 登入流程。 |
| `exit`, `quit` | 結束遊戲 | 終止程式。 |

---

## 陸、Pokey — 模擬 UNIX 環境的電腦

### 登入方式

啟動 computer 後，輸入：

```bash
type
```

即可看到登入畫面。

帳號與密碼可以透過己查 mailroom 的 bin 來得到，分別為：

```plain
login: toukmond
password: robert
```

登入成功後，玩家會進入模擬 UNIX 環境：

```plain
UNIX System V, Release 2.2 (pokey)
```

Pokey 的初始工作目錄為：

```bash
/usr/toukmond
```

### Pokey 的路徑處理

Pokey 不會直接將真實 shell 的工作目錄切換至虛擬路徑，而是使用：

```bash
# current directory in pokey, 
# initialized to "/usr/toukmond" since it's the home directory of toukmond
set pokey_pwd = "/usr/toukmond"
```

記錄玩家在 Pokey 中的位置。

程式再將 Pokey 路徑轉換為真實檔案系統中的位置：

```bash
$game_root/filesystem/pokey
```

處理 `cd` 與 `ls` 時，程式會暫時使用真實的 `cd` 與 `pwd -P` 正規化路徑，處理：

```bash
.
..
重複的 /
```

並阻止玩家離開 Pokey root。

```bash
# pokey root directory in real filesystem, 
# used for path normalization in cd and ls command in pokey
set pokey_root = "$game_root/filesystem/pokey" 
```

---

## 柒、Pokey 指令

| 指令 | 作用 | 實作方式 |
| --- | --- | --- |
| `exit` | 離開 Pokey，回到一般冒險模式 | 顯示離開 console 的訊息，並跳回 `game_loop`。 |
| `echo <text>` | 顯示輸入文字 | 輸出第二個參數之後的所有內容。 |
| `pwd` | 顯示 Pokey 目前路徑 | 顯示 `pokey_pwd`。 |
| `ls` | 顯示目前目錄內容 | 根據 `pokey_pwd` 顯示預先定義的 UNIX-like 輸出 *(硬編碼)*，並動態列出 inventory 或房間內的 `.o` 檔案。 |
| `ls <path>` | 顯示指定目錄內容 | 先將相對路徑或絕對路徑正規化，再顯示對應目錄。 |
| `cd <path>` | 切換 Pokey 工作目錄 | 正規化目標路徑後更新 `pokey_pwd`，不會修改到遊戲本體的真實工作路徑。 |
| `cat <file>` | 顯示 ASCII file 內容 | 僅允許讀取目前資料夾內的部分文字檔案，例如 `description`。 |
| `uncompress` | 解壓縮檔案 | 使用 `gunzip` 解壓縮指定檔案，目前遊戲可解壓縮的檔案為 `paper.o.gz` *(原始 Dunnet 中為`paper.o.Z`)*。 |
| `ftp` | *尚未完成* | *非作業要求，功能尚未實作。* |
| `rlogin` | *尚未完成* | *非作業要求，功能尚未實作。* |
| `ssh` | *尚未完成* | *非作業要求，功能尚未實作。* |

### Pokey 與一般遊戲模式的輸入差異

一般冒險模式會將：

```bash
:
;
,
```

視為 delimiters。

Pokey 則模擬 UNIX shell，因此目前只按照空白切割參數，不應直接套用一般遊戲模式的 delimiter 規則。

---

## 捌、開發過程中遇到的問題與解法

### 1.  被展開成 wildcard

### 問題

當玩家輸入

```bash
x *
```

`*` 可能被外層 shell 當成 filename glob pattern，展開成目前房間內的檔案名稱。

例如，當房間內有：

```plain
lamp.o
shovel.o
```

輸入：

```bash
x *
```

可能被解讀為：

```plain
x lamp.o shovel.o
```

造成程式錯誤地顯示某個物品的資訊。

### 解法

在解析一般遊戲模式的輸入前加入：

```bash
set noglob
```

並在建立 `args_padded` 後恢復原先狀態：

```bash
unset noglob
```

`tcsh` 的 `noglob` 變數會停用 filename substitution，因此：

```bash
*
?
[abc]
```

不會被 shell 展開成檔案名稱。

---

### 2. Dunnet 使用多種 delimiters

### 問題

原始 Dunnet 中，空格、冒號、分號與逗號都會被視為參數分隔符號。

因此：

```bash
x shovel
x:shovel
x;shovel
x,shovel
```

應該具有相同效果。

### 解法

使用 `sed` 將連續 delimiters 統一轉換為一個空格：

```bash
set input = `echo "$input" | sed 's/[ :;,][ :;,]*/ /g'`
```

---

### 3. 空指令造成錯誤

### 問題

在一般遊戲模式或 Pokey 中，如果玩家沒有輸入任何內容，直接按下 Enter，程式可能會在解析參數或存取陣列元素時出錯。

### 解法

讀取輸入時使用：

```bash
# normal input
set input = $<:q

# pokey input
set pokey_cmd = $<:q
```

接著，在存取參數前檢查陣列長度：

```bash
if ( "$#args" == 0 ) goto game_loop

if ( "$#pokey_args" == 0 ) goto pokey_loop
```

`:q` 可以降低輸入被 shell 重新解讀的機會；陣列長度檢查則負責跳過真正的空指令。

兩者用途不同，缺一不可。

---

### 4. 存取不存在的陣列元素

### 問題

在 `csh` 中，如果直接存取不存在的陣列元素，例如：

```bash
$args[3]
```

可能出現：`error: Subscript out of range`

### 解法

建立 padded array：

```bash
set args_padded = ( $args:q "" "" "" "" )
```

之後統一存取：

```bash
$args_padded[1]
$args_padded[2]
$args_padded[3]
$args_padded[4]
```

Pokey 中也使用相同方式：

```bash
set pokey_args_padded = ( $pokey_args:q "" "" "" "" )
```

---

### 5. Symbolic link 造成房間路徑不一致

### 問題

玩家透過方向 link 移動後，logical path 與實體路徑可能不同。

例如：

```bash
e -> ../e-w-dirt-road
```

如果直接使用目前顯示的路徑判斷房間，可能造成重複造訪判斷錯誤。

### 解法

移動後使用：

```bash
pwd -P
```

取得 symbolic link 解析後的 physical path。

Pokey 中的 `cd` 與 `ls` 也透過暫時切換到真實目錄，再使用：

```bash
pwd -P
```

進行路徑正規化。

---

### 6. 同一個物品具有多個名稱

### 問題

CPU board 可被視為：

```plain
board
card
chip
cpu
```

Bracelet 也可被視為：

```plain
bracelet
emerald
```

如果將每個名稱都當成獨立物品，玩家可能重複撿取同一個物品。

### 解法

使用一個主要 `.o` 檔案代表真實物品：

```plain
board.o
bracelet.o
```

---

## 玖、尚未處理或待確認的問題

### 1. Pokey 中的 `ls` 會顯示 `boulder.o`

登入 Pokey 後，切換至：

```bash
/rooms/e-w-dirt-road
```

再輸入：

```bash
ls
```

目前版本會顯示：

```plain
boulder.o
```

但原版 Dunnet 中不會顯示該檔案。

### 原因

目前 Pokey 的 `ls` 邏輯會列出 room 資料夾內所有一般 `.o` 檔案：

```bash
foreach file ( `sh -c "ls -tr $game_root/filesystem/pokey/${target_path}/*.o 2>/dev/null"` )
```

而 `boulder.o` 符合條件，因此會被輸出。

### 待確認事項

目前尚未確認原版 Dunnet 的處理方式。可能原因包括：

1. 原版會額外過濾不可撿取的物品。
2. `boulder` 在原版中不是一般 `.o` 檔案。
3. 原版的 room `ls` 使用另一套白名單邏輯。
