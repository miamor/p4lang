# Heavy hitter detection

## 1. Overview
"heavy hitter" flows là những flows với lượng traffic lớn, việc detect ra những flows kiểu này rất quan trọng trong 1 số trường hợp, ví dụ như phát hiện tấn công dos...   
Tuy nhiên, việc tính toán ở data plane bị giới hạn bởi line-rate processing (~ 10-100Gb/s) và bị giới hạn bộ nhớ (memory accessing).   

Ở ví dụ này ta sẽ sử dụng p4 để code lại thuật toán HashPipe, cho phép track được k heaviest flows.   
HashPipe giữ cả keys và counters của các heavy flows trong switch, xuyên suốt cả pipeline (multiple stages).  

Thuật toán này giúp ta phát hiện được những gói tin trong 1 heavy hitter flow ngay khi nó đang đi qua thiết bị, từ đó thiết bị này có thể lập tức xác định hành động kế tiếp với những gói tin này.   

------------

Khi 1 gói tin được hash ở stage đầu tiên của pipeline, counter của nó được cập nhật (hoặc khởi tạo) nếu có một hit (hay empty slot) (tức là nếu key đó đã có trong bảng hay bảng còn empty slots). Còn nếu là 1 miss (tức là key này chưa có trong bảng và bảng đã đầy), thì key mới sẽ được thêm vào bảng bằng cách thay thế với một key cũ trong bảng, mà giá trị counter không bị reset (chi tiết ở phần Saving Space Algo)   
Ở các stage sau đó, key và counter của item vừa bị thay thế trong bảng được carried kèm theo packet. Carried key được tìm kiếm trong hash table ở stage hiện tại. 
Ở mỗi stage, các carried key sẽ được looked up trong hash table. Và giữa hai giá trị counter, 1 được carried kèm theo packet, và 1 được lưu trong table, giá trị nào **lớn hơn** sẽ được lưu lại trong hash table, và giá trị còn lại, hoặc là được carried kèm theo packet tới những stage tiếp theo, hoặc là remove hoàn toàn khỏi switch nếu packet đang ở stage cuối cùng.  


## 2. Background on heavy-hitter detection
### 2.1. Problem formulation
- **Heavy hitters.** Các flows có lượng packets (or bytes) lớn hơn (tổng số packets *seen on the link*)/t ("threshold-t" problem). Hoặc cũng có thể là top k flows có size lớn nhất ("top-k" problem). Phần còn lại trong bài này sử dụng định nghĩa kiểu "top-k" problem.
- **Flow granularity.** Flows có thể được phân tích tới các mức chi tiết khác nhau, ví dụ thành IP address, transport port number, hay ***five-tuple*** (transport port connection). Do lượng keys sẽ tăng lên, nên sẽ yêu cầu nhiều bits hơn để đại diện cho các key và nhiều entries hơn được thêm vào table để track heavy flows để đảm bảo độ chính xác.   
- **Accuracy.** Một kỹ thuật detect heavy hitters có thể có những kết quả sai như *false positives* (non-heavy classified as heavy), *false negative* (heavy classified as non-heavy) hay có lỗi khi estimate heavy flow size. Ảnh hưởng của các lỗi này như thế nào còn phụ thuộc vào ứng dụng. Ví dụ, nếu switch vẫn hoạt động ổn định được thì một vài kết quả false negatives có thể chấp nhận được. Đặc biệt nếu như những heavy flows này ***có thể phát hiện được ở time interval tiếp theo***. Ví dụ khác, nếu network được config những heavy flows đều bị gán nhãn nghi ngờ là dos attack, thì những false positives lại có thể dẫn tới nhiều alarms bỏ. 
- **Overhead.** Phần overhead của switch bao gồm tổng amount of memory dành cho table, lượng matching stages trong pipeline (***constrained by a switch-specific maximum***) Thuật toán này bị giới hạn bởi memory access cho phép và lượng computation ở mỗi match stage (như là tính toán hash functions). 

### 2.2. Existing Solutions
Skip.


## 3. HashPipe
HashPipe được dựa trên ý tưởng của thuật toán **Space Saving**.
### 3.1. Space saving
Để track được k heaviest items, space saving sử dụng 1 table với m slots, mỗi slot chứa 1 key và counter tương ứng. Các slots được khởi tạo là trống.   
Khi 1 gói tin tới, nếu nó không thuộc flow nào đã có sẵn trong table và table vẫn còn chỗ, 1 row mới được thêm vào bảng với counter đặt bằng 1.
Nếu gói tin này thuộc về flow đã có trong bảng, count của flow này sẽ được update (+1)
Nếu table đã hết chỗ, thuật toán này sẽ thay row có counter nhỏ nhất bằng flow của packet vừa tới, và tăng giá trị counter nhỏ nhất này thêm 1 (không reset lại bằng 1)

Thuật toán này có lợi thế như sau: Giả sử counter thực tế của flow key<sub>j</sub> là c<sub>j</sub>, và counter tương ứng của nó trong bảng là val<sub>j</sub>. 
1. Không flow counter nào trong bảng bị đánh giá thấp (underestimated), vì val<sub>j</sub> >= c<sub>j</sub> (theo nguyên lý đã trình bày ở trên)
2. Giá trị nhỏ nhất trong bảng val<sub>r</sub> là upper bound on the overestimation error (giá trị sai số lớn nhất giữa counter lưu trong bảng và counter thực tế) của 1 row bất kỳ. Tức là val<sub>j</sub> <= c<sub>j</sub> + val<sub>r</sub> 
3. Bất kỳ flow nào với counter thực tế lớn hơn giá trị trung bình của counter trong toàn bảng (ví dụ c<sub>j</sub> > C/m) thì luôn luôn tồn tại trong bảng. (C là tổng counter của các packet trong bảng)

### 3.2. Sampling for the mminimum value
Trước hết thay đổi việc tìm giá trị nhỏ nhất trong cả bảng thay bằng tìm giá trị nhỏ nhất của một số d (d nhỏ, và đã biết) các counters được chọn ngẫu nhiên (algo 1.). Cách làm này sẽ giảm lượng memmory access per packet.   
Và phiên bản modifiy là HashParallel (algo 2.). Cách làm này thay đổi lượng slots được đưa ra xem xét mỗi lần (khi look up hay update một key nào đó). Thay đổi chủ yếu so với Algo 1 là ở set of table slots được xem xét, ở đây sẽ là d slots tính được từ d hash functions độc lập nhau.  
Cách này cho phép chúng ta xem xét và chọn ra (estimate) một giá trị minimum counter một cách tương đối mà phải xem xét ít rows hơn. Tuy nhiên, counter nhỏ nhất của d rows có thể khác xa counter nhỏ nhất của cả bảng.   
Và, dù với algo 2. thì switch vẫn phải đọc d locations một lúc để xác định giá trị min, sau đó write lại giá trị đã được updated. Tức là, sẽ cần d lượt reads và 1 lượt write (vào vị trí nào đó trong bảng), với per-packet time budget.   

### 3.3. Multiple stages of HashTables
Giờ ta cần tìm cách giảm số lượt reads xuống. Ta sẽ chia bảng T thành d bảng, và chỉ xem xét 1 slot cho cả bảng con này (read exactly one slot per table). Cách làm giống như algo 2., chỉ khác là hash function h<sub>i</sub> trả về slot của stage thứ i.   
Cấu trúc này sẽ cho phép các packet khác nhau truy cập được tới các bảng khác nhau cùng 1 lúc. Tuy nhiên, một gói tin có thể phải pass qua pipeline này 2 lần: 1 để xác định minimum value của d slots, và lần 2 là để update counter này. *Lần pass thứ 2 có thể qua "recirculation" của packet qua switch pipeline với metadata được thêm vào packet, cho phép switch tăng cái minimum counter.*  Tuy nhiên, lần pass thứ 2 có thể sẽ phải pass tất cả các packet, và recirculating tất cả các packet có thể ngốn 1 nửa the pipeline bandwidth.   
### 3.3. Feed-forwarding packet processing
Giờ ta sẽ tìm cách giải quyết việc packet phải được xử lý nhiều hơn 1 lần. Có 2 ý tưởng chính:   
**Track a rolling minimum.** Ta đang track minimum counter khi packet qua pipeline bằng cách move counter và key qua pipeline theo dạng packet metadata.  
Khi 1 packet qua pipeline, switch sẽ hash vào mỗi stage sử dụng carried key, thay vì hash dựa trên key corresponding to the incoming packet. Nếu key có trong bảng, hay slot còn trống, thì counter được cập nhật theo cách thông thường, và key không cần được carried theo trong packet nữa. Còn không thì key và counter ứng với counter lớn hơn sẽ được lưu lại vào trong table, cặp giá trị còn lại sẽ tiếp tục được carried kèm theo packet. Với p4, ta sử dụng match-action tables để implement việc so sánh 2 counter.  
**Always insert in the first stage.** Nếu incoming key không có ở first stage, sẽ không có counter nào ứng với key đó được lưu để mà so sánh với counter được carried theo packet cả. Ở đây, ta sẽ thêm new flow vào first stage, và existing key và counter nếu bị loại bỏ sẽ được thêm vào packet metadata. Sau stage này, các packet có thể *track the rolling minimum* của subsequent stages theo cách thông thường.

## 4. Prototype in p4
Register arrays lưu flow identifiers (hashed thành các keys) và counter tương ứng.  
**Hashing to sample locations.** Bước đầu tiên là hash các flow identifier với các hash function. Kết quả được dùng để chọn location để check counter với key tương ứng. Thuật hash sử dụng là (a<sub>i</sub>*x + b<sub>i</sub>)%p (a<sub>i</sub>, b<sub>i</sub> nguyên tố cùng nhau)  
**Registers to read and write flow statistics.** Flows được tracked và updated sử dụng 3 registers: 
- 1 để track flow identifiers
- 1 để track packet count
- 1 để test validity của mỗi index (đã tồn tại chưa)

Kết quả thu được từ hàm hash sử dụng là index để access tới các registers. Sau khi flow identifier được read thì kết quả này được so sánh với identifier kèm theo packet, dựa vào đó để quyết định thêm entry, tăng counter, hay replace entry,...  
**Packet metadata for tracking current minimum.** Giá trị đọc được từ registers được lưu vào packet metadata do p4 ko cho phép kiểm tra điều kiện trực tiếp trên register values.
Từ đó tính toán ra giá trị nhỏ hơn giữa carried key và key được lưu trong table trước khi ghi lại vào register (write into the register).  
**Conditional state updates to store local maximum.** Stage đầu tiên trong pipeline sẽ update register để phân biệt nó là 1 hit hay miss (phần overview). Những stage sau sẽ ghi lại các giá trị key và counter vào registers, dựa vào kết quả sau khi look up (check hit/miss) và kết quả so sánh giữa các flow counters.

### P4 Prototype 
```
action updateHash () {
    mKeyCarried = ipv4.srcAddr;
    mCountCarried = 0;
    /* hash on identifier, results give mIndex (position (offset) to access (read/write) data) */
    modify_field_with_hash_based_offset (mIndex, 0, stage1Hash, 32) ;

    // read the key and value at mIndex
    mKeyTable = flowTracker[mIndex];
    mCountTable = packetCount[mIndex];
    mValid = validBit [mIndex];

    // check for empty location or different key
    mKeyTable = (mValid == 0) ? mKeyCarried : mKeyTable;
    mDif = (mValid == 0) ? 0 : mKeyTable − mKeyCarried;

    // update hash table
    flowTracker[mIndex] = ipv4 . srcAddr;
    packetCount[mIndex] = (mDif == 0) ? mCountTable+1: 1;
    validBit [mIndex] = 1;

    // update metadata carried to the next table stage
    mKeyCarried = (mDif == 0) ? 0 : mKeyTable;
    mCountCarried = (mDif == 0) ? 0 : mCountTable;
}
```


## Implementation
Đây là bài tập của khóa Stanford_CS344_2018 (ko có link online ~"~)  

Trong đoạn code có sử dụng 1 số extern từ v1model.p4:
```
extern register<T>  {
    register(bit<32>  i nstance_count);
    void  read(out T  result,  i n bit<32> idx);
    void  write(in bit<32>  i dx,  in T  value);
}

enum HashAlgorithm {
    crc32,
    crc32_custom,
    crc16,
    crc16_custom,
    random,
    identity,
    csum16,
    xor16
}
extern void hash<O, T, D, M>(out O result, in HashAlgorithm algo, in T base, in D data, in M max);
```

Ở đây họ đã code sẵn một số chỗ, và họ để 1 số phần như là bài tập (TODO), có một số todo đơn giản thì em hiểu, em code được. Nhưng đọc cả project thì em cũng chưa hiểu lắm, cả workflow của nó rất khó hiểu ấy :< em chưa có thấy nó giống với prototype gì cả :~ Hay như lúc nó chia thành các registers hashtable_a với hashtable_b để làm gì?! Hay các kiểu match kind em vẫn chưa rõ hết, như ternary chẳng hạn?!  

Chưa xong nhưng em cũng build lại bằng p4app, sau các proj khác chắc cũng thế, cho đồng bộ luôn.   
`[sudo] p4app run simple_router_teaching.p4`  

Nói chung là tuần này bị fail mất rồi~