# P4 là gì? Data plane, control plane là gì?
- **Control plane** của 1 thiết bị mạng (router hay switch) là tập hợp bất kỳ thứ gì cần thiết để thực hiện việc định tuyến trên thiết bị đó. 
Những gói tin thuộc control plane là những gói tin được thiết bị đó gửi đi, hoặc được gửi tới thiết bị đó.   
- **Data plane (forwarding plane)** là bất kỳ thứ gì chạy qua thiết bị đó. Tức là trong quá trình định tuyến, các gói tin dùng thiết bị này (router or switch) để tới điểm đích mà chúng được gửi tới. Hay thiết bị này chỉ là 1 trung gian để chuyển gói tin đó tới một đích nào đó.  
   
*Eg: Topo: R1 --- R2 --- R3   
Với R1 và R3, routing updates từ 2 router này thuộc về control plane. Với R2, những gói tin này thuộc về data plane.*

P4 là ngôn ngữ giúp lập trình phần data plane này.   
*Ví dụ, R2 được nạp 1 đoạn code p4 để mỗi gói tin đi qua sẽ tách phần header và phần payload, để tính toán lại checksum, giám TTL của header và gộp lại để chuyển tiếp đi.*

# Cấu trúc của p4
## Templates
```
header_type ethernet_t {
    fields {
        ..
    }
}

parser start {
    ...
}

action rewrite_mac(smac) {
	...
}

table send_frame {
    reads {
        standard_metadata.egress_port: exact;
    }
    actions {
        rewrite_mac;
        _drop;
    }
    size: 256;
}

control egress {
    apply(send_frame);
}
...
```

## P4 types
```
typedef bit<48>  macAddr_t;
typedef bit<32>  ip4Addr_t;
header ethernet_t {
    macAddr_t dstAddr;
    macAddr_t srcAddr;
    bit<16>      etherType;
}
header ipv4_t  {
    bit<4>        version;
    bit<4>        ihl;
    bit<8>        diffserv;
    bit<16>      totalLen;
    bit<16>      identification;
    bit<3>        flags;
    bit<13>      fragOffset;
    bit<8>        ttl;
    bit<8>        protocol;
    bit<16>      hdrChecksum;
    ip4Addr_t  srcAddr;
    ip4Addr_t  dstAddr;
}
```

## Parsers
Đây là các hàm tách mỗi gói tin thành 2 phần: headers và metadata.  
Mỗi parser có 3 states được định nghĩa trước:
- start
- accept
- reject

Có thể định nghĩa thêm các state khác.  
Tại mỗi state, có thể có hoặc không thực thi câu lệnh nào, sau đó chuyển tiếp (transition) sang state khác (loops are ok).  
```
parser parse_ipv4 {
    extract(ipv4);
    return ingress;
}
/* or */
parser parse_ethernet {
    extract(ethernet);
    return select(latest.etherType) {
        ETHERTYPE_IPV4 : parse_ipv4;
        default: ingress;
    }
}
/* or */
parser MyParser (packet_in packet,
                 out headers  hdr,
                 inout metadata meta,
                 inout standard_metadata_t std_meta
  ) {
    state start  {
        parser.extract(hdr.ethernet);
        transition accept;
    }
}
```
- Parser sẽ tạo ra 1 “Parsed Representation” (PR) cho mỗi gói tin
- Match+action tables sử dụng PR để match và thực thi action
- Bất kỳ giá trị nào được sử dụng bởi chương trình đều phải có trong PR


### Multi-field select statement
```
parser parse_ipv4 {
    extract(ipv4);
    set_metadata(ipv4_metadata.lkp_ipv4_sa, ipv4.srcAddr);
    set_metadata(ipv4_metadata.lkp_ipv4_da, ipv4.dstAddr);
    set_metadata(l3_metadata.lkp_ip_proto, ipv4.protocol);
    set_metadata(l3_metadata.lkp_ip_ttl, ipv4.ttl);
    /* Fields are joined for a match */
    // return select(latest.etherType) {}
    return select(latest.fragOffset, latest.ihl, latest.protocol) {
        0x0000501 : parse_icmp;
        0x0000506 : parse_tcp;
        0x0000511 : parse_udp;
        default : ingress;
    }
}
```
### Calculated fields
```
header ipv4_t ipv4;
field_list ipv4_checksum_list {
    ipv4.version;
    ipv4.ihl;
    ipv4.diffserv;
    ipv4.totalLen;
    ipv4.identification;
    ipv4.flags;
    ipv4.fragOffset;
    ipv4.ttl;
    ipv4.protocol;
    ipv4.srcAddr;
    ipv4.dstAddr;
}
field_list_calculation ipv4_checksum {
    input {
        ipv4_checksum_list;
    }
    algorithm : csum16;
    output_width : 16;
}
calculated_field ipv4.hdrChecksum {
    verify ipv4_checksum;
    update ipv4_checksum;
}
```

## Controls
- Trong một control có thể khai báo biến, tạo bảng, khởi tạo externs.  
- Có thể sử dụng apply {} để thiết lập các câu lệnh thực thi.  
- Hoặc thực thi các actions bằng cách apply 1 table, không thể apply trực tiếp các actions này. Muốn gọi 1 actions mà không phải match (ko cần điều kiện gì), có thể sử dụng 1 bảng trống (“empty” table)

```
control ingress {
    if (valid(ipv4) and ipv4.ttl > 0) {
        apply(ipv4_lpm);
        apply(forward);
    }
}
/* or */
/* Đổi source và dest MAC address, và chuyển tiếp traffic */
control MyIngress ( inout headers  hdr,
                    inout metadata  meta,
                    inout standard_metadata_t std_meta
)  {
    bit<48>  tmp;
    apply {
        tmp =  hdr.ethernet.dstAddr;
        hdr.ethernet.dstAddr =  hdr.ethernet.srcAddr;
        hdr.ethernet.srcAddr =  tmp;
        std_meta.egress_spec =  std_meta.ingress_port;
        // ingress_port - the port on which the packet arrived
        // egress_spec - the port to which the packet should be sent to
        // egress_port - the port on which the packet is departing from (read only in egress pipeline)
    }
}
```
- Phải thiết lập kết nối giữa Ingress port và egress port để có thể chuyển tiếp network traffic.  
*Eg: Với chế độ “switchport mode access”, sheader info phải bị xóa trước khi chuyển tiếp ra cổng egress port. Với chế độ switchport mode trunk, header info được giữ nguyên.*


## Tables
1 table là 1 đơn vị cơ bản của Match-Action pipeline.  
Tác vụ:
- Match các gói tin với điều kiện (defines what data to match on and match kind).  
- Định nghĩa các actions cho phép (permitted actions).
- (Optional) Định nghĩa 1 số properties.
    - Size
    - Default action
    - Static entries
- Mỗi bảng gồm 1 hoặc nhiều entries (rules). Mỗi entry gồm:
    - 1 key để match
    - 1 (và chỉ 1) action sẽ được thực thi nếu gói tin match với entry này (match on the key)
    - Action data (có thể rỗng)


Tóm lại, 1 bảng có nhiệm vụ map những gói tin đến (incoming packets) (sử dụng PR) thành logical incoming interface.

```
table ipv4_lpm  {
    /* lookup on keys / or can use reads */
    key =  {
        /* lpm is a match kind */
        hdr.ipv4.dstAddr:  lpm;
    }
    /* perform actions on those matched */
    actions =  {
        ipv4_forward;
        drop;
        NoAction;
    }
    /* properties */
    size =  1024;
    default_action =  NoAction();
}
```
Đoạn code trên implement cho rules sau:
| Key           | Action        | Action data                          |
| ------------- | ------------- | ------------------------------------ |
| 10.0.1.1/32   | Ipv4_forward  | dstAddr=00:00:00:00:01:01 <br>port=1 |
| 10.0.1.2/32   | drop          |                                      |
| *`            | NoAction      |                                      |

### Match kinds
Một kiểu dữ liệu đặc biệt của p4, cho phép match các packet theo một cách nào đó   

```
/*****************************
 * The standard library (core.p4) defines three standard match kinds:
 * ◦ Exact  match
 * ◦ Ternary  match
 * ◦ LPM  match  
 ****************************/
match_kind {
    exact,        
    ternary,
    lpm
}

/*****************************
 * The architecture (v1model.p4) defines two additional match kinds:
 * ◦ range
 * ◦ selector
 ****************************/
match_kind {
    range,
    selector
}

/*****************************
 * Other architectures may define
 * (and provide implementation for) 
 * additional match kinds
 ****************************/
match_kind {
    regexp,
    fuzzy
}
```

## Actions
Các actions con trong 1 action cha được thực thi tuần tự   
(các phiên bản trước v1.0.2 thì được thực thi song song)

```
control MyIngress ( inout headers hdr,
                    inout metadata meta,
                    inout standard_metadata_t standard_metadata
) {
    table ipv4_lpm {
        ...
    }
    apply {
        ...
        ipv4_lpm.apply();
        ...
    }
}
```


# Execute p4 program
## Native way
Dependencies: bmv2, p4c-bm
1. Compile p4 program to .json file:
`p4c-bmv2 --json <path to JSON file> <path to P4 file>`
2. Setup veth `[sudo] ./veth_setup.sh`
3. Using bmv2 backend to execute:
`sudo ./simple_switch -i 0@<iface0> -i 1@<iface1> <path to JSON file>`

## Using external libraries:
Install p4c and compile using command:  
```
p4c-bm2-ss -o simple_router.json simple_router.p4
```   
or
```
p4c [-b bmv2-ss-p4org] simple_router.json simple_router.p4 (output is a folder)
    ^ declare backend
```


## Using p4app to compile