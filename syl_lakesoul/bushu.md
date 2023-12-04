> rust cargo build一直出现 Blocking waiting for file lock on package cache
如果确定没有多个程序占用，可以删除`rm -rf ~/.cargo/.package-cache`，然后再执行


## 测试
`cargo +nightly test   --package lakesoul-datafusion -- --nocapture`
不行就
`cargo update`
## 调试
`rust-gdb target/debug/deps/lakesoul_datafusion-2089a0f7751f0a7e`

[Rust:test](https://github.com/lakesoul-io/LakeSoul/blob/main/rust/lakesoul-datafusion/src/test/upsert_tests.rs)
[Spark:test](https://github.com/lakesoul-io/LakeSoul/blob/main/lakesoul-spark/src/test/scala/org/apache/spark/sql/lakesoul/commands/UpsertSuiteBase.scala#L557)

* 测试模块的`#[cfg(test)]` 标注告诉 Rust 只在执行 cargo test 时才编译和运行测试代码，而在运行 cargo build 时不这么做


## ubuntu上部署c++使用arrow
部署失败
* 根本原因：Google 放弃了对 Linux 上 32 位 Chrome 的支持，在 64 位系统中更新 apt 时触发错误（启用了多架构）...详细信息如下：http: //www.omgubuntu.co.uk/2016/ 03/修复无法获取 google-chrome-apt-error-ubuntu

[谷歌后的解决办法](https://askubuntu.com/questions/741410/skipping-acquire-of-configured-file-main-binary-i386-packages-as-repository-x)


## lakesoul reader
* **pub(crate)** 使一个程序项在当前 crate 中可见。

## rust调试经过
如果你想在特定的函数或行设置断点，你应该提供正确的函数名或行号。
`(gdb) b module::my_function`
`(gdb) b lakesoul_datafusion.rs:6`
### 调试#[test]程序
> Rust 的测试通常是在一个单独的二进制文件中运行的，该文件位于 target/debug/ 目录下，其名称通常与测试模块或文件的名称相关。确保在运行调试器时使用正确的二进制文件名。

运行`cargo +nightly test --package lakesoul-datafusion -- --nocapture`
![结果](./picture/rust_picture/rust_debug_1.png)

` Running unittests src/lib.rs (target/debug/deps/lakesoul_datafusion-2089a0f7751f0a7e)`
由此可知二进制文件位置。
`rust-gdb target/debug/deps/lakesoul_datafusion-2089a0f7751f0a7e`使用gdb调试

> 之后想打断点很文件结构与自己项目中的文件结构不同，无法通过list查看源文件来一步步打断点。直接对测试函数打断点。

如`(gdb)b test::upsert_tests::upsert_with_io_config_tests::test_basic_upsert_different_columns`

> * 注异步任务(异步函数)没法单步调试。 

### 调试过程
`upsert.rs` 中代码
```
//24
async fn test_derange_hash_key_and_data_schema_order_int_type_upsert_3_times_i32() -> Result<()>
```

```
expected:

[
    "+----------+-------+-------+-------+------+------+",
    "| range    | hash1 | hash2 | value | name | age  |",
    "+----------+-------+-------+-------+------+------+",
    "| 20201102 | 1     | 12    | 1     | 11   |      |",
    "| 20201102 | 3     | 32    | 3     | 345  | 3456 |",
    "| 20201102 | 4     | 42    | 4     | 456  | 4567 |",
    "+----------+-------+-------+-------+------+------+",
]
actual:

[
    "+----------+-------+-------+-------+------+------+",
    "| range    | hash1 | hash2 | value | name | age  |",
    "+----------+-------+-------+-------+------+------+",
    "| 20201102 | 1     | 12    | 1     | 11   |      |",
    "| 20201102 | 3     | 32    | 3     | 33   |      |",
    "| 20201102 | 4     | 42    | 4     | 456  | 4567 |",
    "+----------+-------+-------+-------+------+------+",
]
```
两个execute_upsert都注释掉，
```
actual:

[
    "++",
    "++",
]
```


check_upsert插入为空时结果

```
actual:

[
    "+----------+-------+-------+-------+------+-----+",
    "| range    | hash1 | hash2 | value | name | age |",
    "+----------+-------+-------+-------+------+-----+",
    "| 20201102 | 1     | 12    | 1     | 11   |     |",
    "| 20201102 | 3     | 32    | 3     | 33   |     |",
    "| 20201102 | 4     | 42    | 4     |      |     |",
    "+----------+-------+-------+-------+------+-----+",
]
```
`execute_upsert`二次插入是成功的

总结：未变动代码代码时check_upsert部分插入成功，注释掉execute_upsert完全没插入进去，插入为空时，前两个execute_upsert插入成功。

注释第二个`execute_upsert`
```
actual:

[
    "+----------+-------+-------+-------+------+------+",
    "| range    | hash1 | hash2 | value | name | age  |",
    "+----------+-------+-------+-------+------+------+",
    "| 20201102 | 1     | 12    | 1     |      |      |",
    "| 20201102 | 3     | 32    | 3     |      |      |",
    "| 20201102 | 4     | 42    | 4     | 456  | 4567 |",
    "+----------+-------+-------+-------+------+------+",
]
```
execute_upsert第一次成功，check_upsert部分插入成功

注释第一个`execute_upsert`
```
actual:

[
    "++",
    "++",
]
```
注释第一个`execute_upsert`和check_upsert插入为空
```
actual:

[
    "++",
    "++",
]
```
怀疑是`execute_upsert`时的schema问题导致的。
经测试不是schema的问题。
又怀疑是fliter的问题。（未证明）

* 改写check_upsert进行打印测试
```
check_upsert_print(
            table_name, 
            vec!["range", "hash1", "hash2", "value", "name", "age"],
            // Some("and(and(noteq(range, null), eq(range, 20201102)), noteq(value, null))".to_string()),
            None,
            client.clone(), 
            &[
                "+----------+-------+-------+-------+------+-----+",
                "| range    | hash1 | hash2 | value | name | age |",
                "+----------+-------+-------+-------+------+-----+",
                "| 20201101 | 1     | 1     | 1     | 1    | 1   |",
                "| 20201101 | 2     | 2     | 2     | 2    | 2   |",
                "+----------+-------+-------+-------+------+-----+",
            ]
        ).await?;
```

* 经过四次插入打印发现是第四次插入错误
```
expected:

[
    "+----------+-------+-------+-------+------+------+",
    "| range    | hash1 | hash2 | value | name | age  |",
    "+----------+-------+-------+-------+------+------+",
    "| 20201101 | 1     | 1     | 1     | 1    | 1    |",
    "| 20201101 | 2     | 2     | 2     | 2    | 2    |",
    "| 20201102 | 1     | 12    | 1     | 11   |      |",
    "| 20201102 | 2     | 22    |       | 234  | 2345 |",
    "| 20201102 | 3     | 32    | 3     | 345  | 3456 |",
    "| 20201102 | 4     | 42    | 4     | 456  | 4567 |",
    "+----------+-------+-------+-------+------+------+",
]
actual:

[
    "+----------+-------+-------+-------+------+------+",
    "| range    | hash1 | hash2 | value | name | age  |",
    "+----------+-------+-------+-------+------+------+",
    "| 20201101 | 1     | 1     | 1     | 1    | 1    |",
    "| 20201101 | 2     | 2     | 2     | 2    | 2    |",
    "| 20201102 | 1     | 12    | 1     | 11   |      |",
    "| 20201102 | 2     | 22    |       | 22   |      |",
    "| 20201102 | 3     | 32    | 3     | 33   |      |",
    "| 20201102 | 4     | 42    | 4     | 456  | 4567 |",
    "| 20201102 | 2     | 22    |       | 234  | 2345 |",
    "| 20201102 | 3     | 32    |       | 345  | 3456 |",
    "+----------+-------+-------+-------+------+------+",
]
```
所以是execute_upsert的问题，再此之前先打开写入文件，进行物理确认数据具体样子。
* 文件位置查找
`init_table`中`create_table`中`table_path: [env::temp_dir().to_str().unwrap(), table_name].iter().collect::<PathBuf>().to_str().unwrap().to_string(),`
可知文件具体在
`/tmp/derange_hash_key_and_data_schema_order_int_type_upsert_3_times_i32`中
文件内容
```
1698230168203.parquet#1  1699588716118.parquet  1700121860950.parquet  1700535844312.parquet  1700641382246.parquet  1700648428689.parquet  1700726047155.parquet  1700806191713.parquet  1701056584039.parquet  1701139510058.parquet
1698230570303.parquet#1  1699588716123.parquet  1700121860958.parquet  1700535844316.parquet  1700641382253.parquet  1700710348394.parquet  1700793625205.parquet  1700806792994.parquet  1701056584175.parquet  1701139510326.parquet
1698230901454.parquet    1699588968964.parquet  1700124121913.parquet  1700536822685.parquet  1700641484322.parquet  1700710348407.parquet  1700793625216.parquet  1700806793261.parquet  1701056584439.parquet  1701139510598.parquet
1698230901465.parquet    1699588968975.parquet  1700124121926.parquet  1700536822698.parquet  1700641484332.parquet  1700710348412.parquet  1700793625223.parquet  1700806793527.parquet  1701056584579.parquet  1701139566316.parquet
1698230901470.parquet    1699588968980.parquet  1700124121933.parquet  1700536822705.parquet  1700641484339.parquet  1700710348419.parquet  1700793625230.parquet  1700806793800.parquet  1701056584847.parquet  1701139566587.parquet
1698230901476.parquet#1  1699588968986.parquet  1700124121940.parquet  1700536822711.parquet  1700641484346.parquet  1700719768752.parquet  1700794105686.parquet  1700806834764.parquet  1701056584987.parquet  1701139566853.parquet
1698231065510.parquet    1699596410354.parquet  1700126341039.parquet  1700621099416.parquet  1700641499129.parquet  1700719768767.parquet  1700794105702.parquet  1700806835029.parquet  1701056661829.parquet  1701139682130.parquet
1698231065524.parquet    1699596410382.parquet  1700126341050.parquet  1700621099430.parquet  1700641499396.parquet  1700719768776.parquet  1700794105708.parquet  1700806835291.parquet  1701056661968.parquet  1701139682408.parquet
1698231065528.parquet    1699596410443.parquet  1700126341059.parquet  1700621099436.parquet  1700641499656.parquet  1700719768788.parquet  1700794105717.parquet  1700806835553.parquet  1701056662240.parquet  1701139682675.parquet
1698231065533.parquet#1  1699596410451.parquet  1700126341064.parquet  1700621099441.parquet  1700641501918.parquet  1700719969864.parquet  1700794142400.parquet  1700807050550.parquet  1701056662378.parquet  1701139724725.parquet
1698231340061.parquet    1699602355103.parquet  1700127448894.parquet  1700624343733.parquet  1700642939965.parquet  1700719969875.parquet  1700794142416.parquet  1700807050814.parquet  1701056662711.parquet  1701139724993.parquet
1698231340072.parquet    1699602355115.parquet  1700127448906.parquet  1700624343746.parquet  1700642939976.parquet  1700719969882.parquet  1700794142429.parquet  1700807051077.parquet  1701056662851.parquet  1701139725261.parquet
1698231340079.parquet    1699602355124.parquet  1700127448913.parquet  1700624343751.parquet  1700642939982.parquet  1700719969890.parquet  1700794142441.parquet  1700807051340.parquet  1701056663122.parquet  1701139725537.parquet
1698231340085.parquet#1  1699602355134.parquet  1700127448919.parquet  1700624343764.parquet  1700642939988.parquet  1700720181189.parquet  1700794172510.parquet  1700807072851.parquet  1701056663267.parquet  1701140170791.parquet
1698231471424.parquet    1699868566279.parquet  1700127665123.parquet  1700633751432.parquet  1700642955183.parquet  1700720181200.parquet  1700794172519.parquet  1700807073116.parquet  1701056709842.parquet  1701140171057.parquet
1698231471436.parquet    1699868566291.parquet  1700127665137.parquet  1700633751443.parquet  1700642955669.parquet  1700720181207.parquet  1700794172526.parquet  1700807073380.parquet  1701056709982.parquet  1701140171325.parquet
1698231471442.parquet    1699868566296.parquet  1700127665143.parquet  1700633751448.parquet  1700642955671.parquet  17007201......
```

正常文本编辑器无法查看，所以需要特殊工具。
python工具`parquet-tools` [here](https://pypi.org/project/parquet-tools/)
pip install遇到错误
`Defaulting to user installation because normal site-packages is not writeable`
解决：
`python -m site --user-site`查看`site-packages`位置，尝试修改文件夹权限**发现没用。。。**。
[here](https://itsmycode.com/solved-defaulting-to-user-installation-because-normal-site-packages-is-not-writeable/)**没尝试不知道可以不**

最终决定在本地（自己电脑）python下载尝试。
* 传文件到本地
` scp -i C:\Users\翟爽\.ssh\syl_zhaishuang_rsa zhaishuang@122.9.131.129:/tmp/derange_hash_key_and_data_schema_order_int_type_upsert_3_times_i32/1701151985574.parquet D:\研究生学习\实习单位资料\北京数元灵\resource`
* 进行打印查看
` parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource`
```
PS C:\Users\翟爽> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource
+-------------+-------+---------+--------+---------+
|       range |   age |   hash2 |   name |   hash1 |
|-------------+-------+---------+--------+---------|
| 2.02011e+07 |  4567 |      42 |    456 |       4 |
| 2.02011e+07 |  2345 |      22 |    234 |       2 |
| 2.02011e+07 |  3456 |      32 |    345 |       3 |
+-------------+-------+---------+--------+---------+
```
这看不出来什么东西，可能要读好几个文件

* 执行测试前
```
1701151738042.parquet
1699587316441.parquet    1699944755687.parquet  1700130032607.parquet  1700639876728.parquet  1700645798939.parquet  1700725523143.parquet  1700797667554.parquet  1701056023293.parquet  1701082045937.parquet  1701151738589.parquet
1699587316452.parquet    1699944755699.parquet  1700130032621.parquet  1700639876736.parquet  1700645798945.parquet  1700725523150.parquet  1700797667560.parquet  1701056023559.parquet  1701082046201.parquet  1701151739155.parquet
1699587316459.parquet    1699944755705.parquet  1700130032627.parquet  1700639876752.parquet  1700645798950.parquet  1700725523155.parquet  1700797667567.parquet  1701056023822.parquet  1701082121077.parquet  1701151983878.parquet
1699587316464.parquet    1699944755710.parquet  1700130032633.parquet  1700639876756.parquet  1700645813631.parquet  1700725686981.parquet  1700798598397.parquet  1701056024090.parquet  1701082121349.parquet  1701151984441.parquet
1699587489117.parquet    1699944792215.parquet  1700130133447.parquet  1700641152511.parquet  1700645813894.parquet  1700725686994.parquet  1700798598669.parquet  1701056096468.parquet  1701082121613.parquet  1701151984998.parquet
1699587489132.parquet    1699944792228.parquet  1700130133456.parquet  1700641152522.parquet  1700645814170.parquet  1700725687007.parquet  1700798598938.parquet  1701056096740.parquet  1701082192447.parquet  1701151985574.parquet
1699587489137.parquet    1699944792234.parquet  1700130133462.parquet  1700641152528.parquet  1700645814448.parquet  1700725687024.parquet  1700798599205.parquet  1701056097018.parquet  1701082192720.parquet
1699587489143.parquet    1699944792240.parquet  1700130133468.parquet  1700641152533.parquet  1700645867670.parquet  1700725862697.parquet  1700805836437.parquet  1701056097289.parquet  1701082192987.parquet
1699587580426.parquet    1699945059276.parquet  1700130203448.parquet  1700641313109.parquet  1700645867680.parquet  1700725862709.parquet  1700805836706.parquet  1701056310435.parquet  1701139323445.parquet
1699587580438.parquet    1699945059287.parquet  1700130203461.parquet  1700641313117.parquet  1700645867688.parquet  1700725862715.parquet  1700805836974.parquet  1701056310709.parquet  1701139323714.parquet
1699587580445.parquet    1699945059292.parquet  1700130203466.parquet  1700641313122.parquet  1700645867695.parquet  1700725862725.parquet  1700805837242.parquet  1701056310980.parquet  1701139323978.parquet
1699587580450.parquet    1699945059296.parquet  1700130203472.parquet  1700641313127.parquet  1700648428664.parquet  1700726047132.parquet  1700806190908.parquet  1701056311245.parquet  1701139439901.parquet
1699588716100.parquet    1700121860929.parquet  1700535844288.parquet  1700641382226.parquet  1700648428676.parquet  1700726047143.parquet  1700806191184.parquet  1701056583164.parquet  1701139440169.parquet
1699588716112.parquet    1700121860943.parquet  1700535844306.parquet  1700641382240.parquet  1700648428682.parquet  1700726047148.parquet  1700806191447.parquet  1701056583325.parquet  1701139440436.parquet
```
执行后
```
699587489143.parquet    1700121860943.parquet  1700536822711.parquet  1700642939976.parquet  1700720322699.parquet  1700795690776.parquet  1700818156457.parquet  1701081760361.parquet  1701151738042.parquet
1699587580426.parquet    1700121860950.parquet  1700621099416.parquet  1700642939982.parquet  1700720322712.parquet  1700795690791.parquet  1701053938106.parquet  1701081760637.parquet  1701151738589.parquet
1699587580438.parquet    1700121860958.parquet  1700621099430.parquet  1700642939988.parquet  1700720322719.parquet  1700796041157.parquet  1701053938372.parquet  1701081760899.parquet  1701151739155.parquet
1699587580445.parquet    1700124121913.parquet  1700621099436.parquet  1700642955183.parquet  1700720322724.parquet  1700796041168.parquet  1701054025215.parquet  1701082012761.parquet  1701151983878.parquet
1699587580450.parquet    1700124121926.parquet  1700621099441.parquet  1700642955669.parquet  1700720338841.parquet  1700796041175.parquet  1701054025729.parquet  1701082013024.parquet  1701151984441.parquet
1699588716100.parquet    1700124121933.parquet  1700624343733.parquet  1700642955671.parquet  1700720339110.parquet  1700796041193.parquet  1701054025999.parquet  1701082013292.parquet  1701151984998.parquet
1699588716112.parquet    1700124121940.parquet  1700624343746.parquet  1700642957932.parquet  1700720339405.parquet  1700796390298.parquet  1701054026267.parquet  1701082045667.parquet  1701151985574.parquet
1699588716118.parquet    1700126341039.parquet  1700624343751.parquet  1700643341117.parquet  1700720339692.parquet  1700796390311.parquet  1701054734121.parquet  1701082045937.parquet  1701161852278.parquet
1699588716123.parquet    1700126341050.parquet  1700624343764.parquet  1700643341129.parquet  1700725261429.parquet  1700796390318.parquet  1701054734841.parquet  1701082046201.parquet  1701161852805.parquet
1699588968964.parquet    1700126341059.parquet  1700633751432.parquet  1700643341139.parquet  1700725261440.parquet  1700796390324.parquet  1701054735154.parquet  1701082121077.parquet  1701161853342.parquet
1699588968975.parquet    1700126341064.parquet  1700633751443.parquet  1700643341146.parquet  1700725261446.parquet  1700796626124.parquet  1701054735721.parquet  1701082121349.parquet  1701161853859.parquet
1699588968980.parquet    1700127448894.parquet  1700633751448.parquet  1700643356295.parquet  1700725261452.parquet  1700796626131.parquet  1701055202774.parquet  1701082121613.parquet
1699588968986.parquet    1700127448906.parquet  1700633751453.parquet  1700643356558.parquet  1700725275504.parquet  1700796626136.parquet  1701055203046.parquet  1701082192447.parquet
```

执行一次多了四个文件
`1701161852278.parquet`
`1701161852805.parquet`
`1701161853342.parquet`
`1701161853859.parquet`
传到本地
`scp -i C:\Users\翟爽\.ssh\syl_zhaishuang_rsa zhaishuang@122.9.131.129:/tmp/derange_hash_key_and_data_schema_order_int_type_upsert_3_times_i32/{1701161852278.parquet,1701161852805.parquet,1701161853342.parquet,1701161853859.parquet} D:\研究生学习\实习单位资料\北京数元灵\resource`

* 打印结果
```
PS C:\Users\翟爽> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource
+-------------+---------+---------+---------+--------+-------+
|       range |   hash1 |   hash2 |   value |   name |   age |
|-------------+---------+---------+---------+--------+-------|
| 2.02011e+07 |       1 |       1 |       1 |      1 |     1 |
| 2.02011e+07 |       2 |       2 |       2 |      2 |     2 |
| 2.02011e+07 |       1 |      12 |       1 |    nan |   nan |
| 2.02011e+07 |       3 |      32 |       3 |    nan |   nan |
| 2.02011e+07 |       4 |      42 |       4 |    nan |   nan |
| 2.02011e+07 |       1 |      12 |     nan |     11 |   nan |
| 2.02011e+07 |       2 |      22 |     nan |     22 |   nan |
| 2.02011e+07 |       3 |      32 |     nan |     33 |   nan |
| 2.02011e+07 |       4 |      42 |     nan |    456 |  4567 |
| 2.02011e+07 |       2 |      22 |     nan |    234 |  2345 |
| 2.02011e+07 |       3 |      32 |     nan |    345 |  3456 |
+-------------+---------+---------+---------+--------+-------+
```
> 分别打印
* 1
```
PS C:\Users\翟爽> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource\1701161852278.parquet
+-------------+---------+---------+---------+--------+-------+
|       range |   hash1 |   hash2 |   value |   name |   age |
|-------------+---------+---------+---------+--------+-------|
| 2.02011e+07 |       1 |       1 |       1 |      1 |     1 |
| 2.02011e+07 |       2 |       2 |       2 |      2 |     2 |
+-------------+---------+---------+---------+--------+-------+
```
* 2
```
PS C:\Users\翟爽> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource\1701161852805.parquet
+-------------+---------+---------+---------+
|       range |   hash1 |   hash2 |   value |
|-------------+---------+---------+---------|
| 2.02011e+07 |       1 |      12 |       1 |
| 2.02011e+07 |       3 |      32 |       3 |
| 2.02011e+07 |       4 |      42 |       4 |
+-------------+---------+---------+---------+
```

* 3
```
PS C:\Users\翟爽> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource\1701161853342.parquet
+-------------+---------+--------+---------+
|       range |   hash2 |   name |   hash1 |
|-------------+---------+--------+---------|
| 2.02011e+07 |      12 |     11 |       1 |
| 2.02011e+07 |      22 |     22 |       2 |
| 2.02011e+07 |      32 |     33 |       3 |
+-------------+---------+--------+---------+
```
* 4
```
PS C:\Users\翟爽> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource\1701161853859.parquet
+-------------+-------+---------+--------+---------+
|       range |   age |   hash2 |   name |   hash1 |
|-------------+-------+---------+--------+---------|
| 2.02011e+07 |  4567 |      42 |    456 |       4 |
| 2.02011e+07 |  2345 |      22 |    234 |       2 |
| 2.02011e+07 |  3456 |      32 |    345 |       3 |
+-------------+-------+---------+--------+---------+
```
看出四次创建的文件都没问题，那么问题可能在`merge on read`上。

* 看下之前builder时测试产生的文件
```
1699587316441.parquet    1699944792240.parquet  1700535844312.parquet  1700641499396.parquet  1700720181200.parquet  1700795690755.parquet  1700818156186.parquet  1701081760361.parquet  1701151738589.parquet
1699587316452.parquet    1699945059276.parquet  1700535844316.parquet  1700641499656.parquet  1700720181207.parquet  1700795690765.parquet  1700818156457.parquet  1701081760637.parquet  1701151739155.parquet
1699587316459.parquet    1699945059287.parquet  1700536822685.parquet  1700641501918.parquet  1700720181221.parquet  1700795690776.parquet  1701053938106.parquet  1701081760899.parquet  1701151983878.parquet
1699587316464.parquet    1699945059292.parquet  1700536822698.parquet  1700642939965.parquet  1700720322699.parquet  1700795690791.parquet  1701053938372.parquet  1701082012761.parquet  1701151984441.parquet
1699587489117.parquet    1699945059296.parquet  1700536822705.parquet  1700642939976.parquet  1700720322712.parquet  1700796041157.parquet  1701054025215.parquet  1701082013024.parquet  1701151984998.parquet
1699587489132.parquet    1700121860929.parquet  1700536822711.parquet  1700642939982.parquet  1700720322719.parquet  1700796041168.parquet  1701054025729.parquet  1701082013292.parquet  1701151985574.parquet
1699587489137.parquet    1700121860943.parquet  1700621099416.parquet  1700642939988.parquet  1700720322724.parquet  1700796041175.parquet  1701054025999.parquet  1701082045667.parquet  1701161852278.parquet
1699587489143.parquet    1700121860950.parquet  1700621099430.parquet  1700642955183.parquet  1700720338841.parquet  1700796041193.parquet  1701054026267.parquet  1701082045937.parquet  1701161852805.parquet
1699587580426.parquet    1700121860958.parquet  1700621099436.parquet  1700642955669.parquet  1700720339110.parquet  1700796390298.parquet  1701054734121.parquet  1701082046201.parquet  1701161853342.parquet
1699587580438.parquet    1700124121913.parquet  1700621099441.parquet  1700642955671.parquet  1700720339405.parquet  1700796390311.parquet  1701054734841.parquet  1701082121077.parquet  1701161853859.parquet
1699587580445.parquet    1700124121926.parquet  1700624343733.parquet  1700642957932.parquet  1700720339692.parquet  1700796390318.parquet  1701054735154.parquet  1701082121349.parquet  1701312209323.parquet
1699587580450.parquet    1700124121933.parquet  1700624343746.parquet  1700643341117.parquet  1700725261429.parquet  1700796390324.parquet  1701054735721.parquet  1701082121613.parquet  1701312209341.parquet
1699588716100.parquet    1700124121940.parquet  1700624343751.parquet  1700643341129.parquet  1700725261440.parquet  1700796626124.parquet  1701055202774.parquet  1701082192447.parquet  1701312209347.parquet
1699588716112.parquet    1700126341039.parquet  1700624343764.parquet  1700643341139.parquet  1700725261446.parquet  1700796626131.parquet  1701055203046.parquet  1701082192720.parquet  1701312209353.parquet
1699588716118.parquet    1700126341050.parquet  1700633751432.parquet  1700643341146.parquet  1700725261452.parquet  1700796626136.parquet  1701056023293.parquet  1701082192987.parquet
1699588716123.parquet    1700126341059.parquet  1700633751443.parquet  1700643356295.parquet  1700725275504.parquet  1700796
```

`1701312209323.parquet,1701312209341.parquet，1701312209347.parquet，1701312209353.parquet`
* 传文件到本地
`scp -i C:\Users\翟爽\.ssh\syl_zhaishuang_rsa zhaishuang@122.9.131.129:/tmp/derange_hash_key_and_data_schema_order_int_type_upsert_3_times_i32/1701312209323.parquet 1701312209341.parquet 1701312209347.parquet 1701312209353.parquet D:\研究生学习\实习单位资料\北京数元灵\resource\builder`
scp传送多文件有问题只能一个一个传了
* 打印结果
```
PS D:\研究生学习\文档总结> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource\builder\
+-------------+---------+---------+---------+--------+-------+
|       range |   hash1 |   hash2 |   value |   name |   age |
|-------------+---------+---------+---------+--------+-------|
| 2.02011e+07 |       1 |       1 |       1 |      1 |     1 |
| 2.02011e+07 |       2 |       2 |       2 |      2 |     2 |
| 2.02011e+07 |       1 |      12 |       1 |    nan |   nan |
| 2.02011e+07 |       3 |      32 |       3 |    nan |   nan |
| 2.02011e+07 |       4 |      42 |       4 |    nan |   nan |
| 2.02011e+07 |       1 |      12 |     nan |     11 |   nan |
| 2.02011e+07 |       2 |      22 |     nan |     22 |   nan |
| 2.02011e+07 |       3 |      32 |     nan |     33 |   nan |
| 2.02011e+07 |       2 |      22 |     nan |    234 |  2345 |
| 2.02011e+07 |       3 |      32 |     nan |    345 |  3456 |
| 2.02011e+07 |       4 |      42 |     nan |    456 |  4567 |
+-------------+---------+---------+---------+--------+-------+
```
* print1
```
PS D:\研究生学习\文档总结> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource\builder\1701312209323.parquet
+-------------+---------+---------+---------+--------+-------+
|       range |   hash1 |   hash2 |   value |   name |   age |
|-------------+---------+---------+---------+--------+-------|
| 2.02011e+07 |       1 |       1 |       1 |      1 |     1 |
| 2.02011e+07 |       2 |       2 |       2 |      2 |     2 |
+-------------+---------+---------+---------+--------+-------+
```
* print2
```
PS D:\研究生学习\文档总结> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource\builder\1701312209341.parquet
+-------------+---------+---------+---------+
|       range |   hash1 |   hash2 |   value |
|-------------+---------+---------+---------|
| 2.02011e+07 |       1 |      12 |       1 |
| 2.02011e+07 |       3 |      32 |       3 |
| 2.02011e+07 |       4 |      42 |       4 |
+-------------+---------+---------+---------+
```
* print3
```
PS D:\研究生学习\文档总结> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource\builder\1701312209347.parquet
+-------------+---------+--------+---------+
|       range |   hash2 |   name |   hash1 |
|-------------+---------+--------+---------|
| 2.02011e+07 |      12 |     11 |       1 |
| 2.02011e+07 |      22 |     22 |       2 |
| 2.02011e+07 |      32 |     33 |       3 |
+-------------+---------+--------+---------+
```
* print4
```
PS D:\研究生学习\文档总结> parquet-tools show  D:\研究生学习\实习单位资料\北京数元灵\resource\builder\1701312209353.parquet
+-------------+-------+---------+--------+---------+
|       range |   age |   hash2 |   name |   hash1 |
|-------------+-------+---------+--------+---------|
| 2.02011e+07 |  2345 |      22 |    234 |       2 |
| 2.02011e+07 |  3456 |      32 |    345 |       3 |
| 2.02011e+07 |  4567 |      42 |    456 |       4 |
+-------------+-------+---------+--------+---------+
```
最后发现错误原因是插入的batch的hash key要从小到大有序
```
expected:

[
    "+----------+-------+-------+-------+------+------+",
    "| range    | hash1 | hash2 | value | name | age  |",
    "+----------+-------+-------+-------+------+------+",
    "| 20201101 | 1     | 1     | 1     | 1    | 1    |",
    "| 20201101 | 2     | 2     | 2     | 2    | 2    |",
    "| 20201102 | 1     | 12    | 1     | 11   |      |",
    "| 20201102 | 2     | 22    |       | 234  | 2345 |",
    "| 20201102 | 3     | 32    | 3     | 345  | 3456 |",
    "| 20201102 | 4     | 42    | 4     | 456  | 4567 |",
    "+----------+-------+-------+-------+------+------+",
]
actual:

[
    "+----------+-------+-------+-------+------+------+",
    "| range    | hash1 | hash2 | value | name | age  |",
    "+----------+-------+-------+-------+------+------+",
    "| 20201101 | 1     | 1     | 1     | 1    | 1    |",
    "| 20201101 | 2     | 2     | 2     | 2    | 2    |",
    "| 20201102 | 1     | 12    | 1     | 11   |      |",
    "| 20201102 | 2     | 22    |       | 456  | 4567 |",
    "| 20201102 | 3     | 32    | 3     | 234  | 2345 |",
    "| 20201102 | 4     | 42    | 4     | 345  | 3456 |",
    "+----------+-------+-------+-------+------+------+",
]

```