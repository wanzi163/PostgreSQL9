# BootStrap 模式下的pg_filenode.map文件分析

[TOC]

## 简介

文章来自于关于pg_filenode.map文件内容讨论的内部邮件。本文只涉及到Bootstrap模式下的pg_filenode.map文件写入逻辑。

> Sam,
>
> 我最近在研究BootStrap模式的执行过程，所以也顺便看了下这个模式下的pg_filenode.map文件的写入过程，此邮件只是写了代码中如何生成filenode.map的数据的，至于为什么要用这个文件，如何使用，这里没有进行研究，希望能够帮助你完整分析这个文件的作用和使用过程。邮件中涉及到了一个工具，放在了http://47.90.6.240:8091/files/tools/filemap_print  这个链接上，用法就是直接在pg_filenode.map相同目录下运行即可。

 

## pg_filenode.map文件生成机制

举例下面的在postgres.bki文件中的两个创建系统表的语句：

create pg_proc 1255 bootstrap rowtype_oid 81

create pg_database 1262 shared_relation rowtype_oid 1248

 

两个重要的参数bootstap和shared_relation参数，这两个参数中只要有一个为“真”，那么就会把这个表的相关映射信息写入到"pg_filenode.map"文件中， 

同时pg_filenode.map拥有两种类型，一种是local mapping，一种是sharedmapping。

上面创建表的语句中如果提供了shared_relation参数的话，那么就会是shared类型的mapping，保存在./global/pg_filenode.map文件中。

相反，没有提供shared_relation参数的话，就是local类型的mapping，保存到./base/1/pg_filenode.map文件中。其中1标识当前使用的数据库的oid(当前标识template1数据库)。

 

## pg_filenode.map数据结构

写入的文件内容就是下面的数据结构的二进制数据，MAX_MAPPINGS最大的mapping记录数62。

```c
typedef struct RelMapFile {
    int32      magic; 
    int32       num_mappings;  
    RelMapping  mappings[MAX_MAPPINGS];
    pg_crc32c   crc;   
    int32      pad;  
} RelMapFile;

```

 这个结构中最关键的就是按个RelMapping类型的数组了，RelMapping结构也很简答， 就是两个oid的配对，如下：

```c
typedef struct RelMapping {
    Oid        mapoid;
    Oid        mapfilenode;
} RelMapping;
```

 

## BootStrap模式下的local filenode.map的写入过程

在bootparse.y文件中所有的表格创建的过程都是使用了InvalidOid的filenode作为输入，也就是说在创建起初，每一个表都没有相应的filenode，意味着这个filenode会在后面的后一个环节中赋值，实际上也就是src/backend/catalog/heap.c文件的heap_create()函数中，把要创建的表格的oid付给了filenode，此时filenode和表的oid应该是相同的值，并且把这个匹配对放到上面提到的数据结构中。下面是使用工具打印出的base/1/pg_filenode.map文件的信息

```
for local mappings
there are 15 maps entries
1259 -> 1259
1249 -> 1249
1255 -> 1255
1247 -> 1247
2836 -> 2836
2837 -> 2837
2658 -> 2658
2659 -> 2659
2662 -> 2662
2663 -> 2663
3455 -> 3455
2690 -> 2690
2691 -> 2691
2703 -> 2703
2704 -> 2704
```

对照到postgres.bki文件中的创建表的语句：

```
create pg_proc 1255bootstrap rowtype_oid 81
create pg_type 1247 bootstraprowtype_oid 71
create pg_attribute 1249bootstrap without_oids rowtype_oid 75
create pg_class 1259bootstrap rowtype_oid 83
```



出了上面四个表格的mapping记录之外，其他的都是建立在这四个表格上面的索引的oid对照表。

```
declare toast 2836 2837 onpg_proc
declare toast 2836 2837 onpg_proc
declare unique indexpg_attribute_relid_attnam_index 2658 on pg_attribute using btree(attrelidoid_ops, attname name_ops)
declare unique indexpg_attribute_relid_attnum_index 2659 on pg_attribute using btree(attrelidoid_ops, attnum int2_ops)
declare unique indexpg_class_oid_index 2662 on pg_class using btree(oid oid_ops)
declare unique indexpg_class_relname_nsp_index 2663 on pg_class using btree(relname name_ops,relnamespace oid_ops)
declare indexpg_class_tblspc_relfilenode_index 3455 on pg_class using btree(reltablespaceoid_ops, relfilenode oid_ops)
declare unique indexpg_proc_oid_index 2690 on pg_proc using btree(oid oid_ops)
declare unique indexpg_proc_proname_args_nsp_index 2691 on pg_proc using btree(proname name_ops,proargtypes oidvector_ops, pronamespace oid_ops)
declare unique indexpg_type_oid_index 2703 on pg_type using btree(oid oid_ops)
declare unique indexpg_type_typname_nsp_index 2704 on pg_type using btree(typname name_ops,typnamespace oid_ops)
```

可以看出来整个local 的filenode mapping都是围绕上面的四个表格所创建。另外有一步做了这四个表的mapping记录更新操作，如下，但是实际上数据跟上面的是相同的（如下语句），暂时没看出来有何用处，留在后续研究。

```c
/*bootstrap.c::BootstrapModeMain()->postinit.c::InitPostgres()->relcache.c::relationCacheInitializePhase3()*/
formrdesc("pg_class", RelationRelation_Rowtype_Id, false,true, Natts_pg_class, Desc_pg_class);
formrdesc("pg_attribute", AttributeRelation_Rowtype_Id, false, false, Natts_pg_attribute, Desc_pg_attribute);
formrdesc("pg_proc", ProcedureRelation_Rowtype_Id, false, true, Natts_pg_proc, Desc_pg_proc);
formrdesc("pg_type", TypeRelation_Rowtype_Id, false, true, Natts_pg_type, Desc_pg_type);


/*在relcache.c::formrdesc()函数中*/

if(IsBootstrapProcessingMode())
{
    RelationMapUpdateMap(RelationGetRelid(relation), RelationGetRelid(relation), isshared, true);
}
```

 

## BootStrap模式下的Global filenode.map

原理实际上和local下的文件相同，下面是通过工具解析出来的global/pg_filenode.map文件的内容：

```
for global mappings
there are 32 maps entries
1262 -> 13291
2964 -> 2964
1213 -> 1213
1136 -> 1136
1260 -> 1260
1261 -> 1261
1214 -> 1214
2396 -> 2396
6000 -> 6000
3592 -> 3592
2846 -> 2846
2847 -> 2847
2966 -> 2966
2967 -> 2967
4060 -> 4060
4061 -> 4061
2676 -> 2676
2677 -> 2677
2694 -> 2694
2695 -> 2695
2671 -> 13293
2672 -> 13294
2397 -> 2397
1137 -> 1137
1232 -> 1232
1233 -> 1233
2697 -> 2697
2698 -> 2698
2965 -> 2965
3593 -> 3593
6001 -> 6001
6002 -> 6002
```



下面是相应的postgres.bki文件中创建表的命令

```
create pg_database 1262shared_relation rowtype_oid 1248
create pg_db_role_setting2964 shared_relation without_oids
create pg_tablespace 1213shared_relation
create pg_pltemplate 1136shared_relation without_oids
create pg_authid 1260shared_relation rowtype_oid 2842
create pg_auth_members 1261shared_relation without_oids rowtype_oid 2843
create pg_shdepend 1214shared_relation without_oids
create pg_shdescription2396 shared_relation without_oids
createpg_replication_origin 6000 shared_relation without_oids
create pg_shseclabel 3592shared_relation without_oids
```

上面的记录中pg_database这个表有些特殊，他的filenode值为13291，以及建立在这个表上面的索引的filenode也跟其他的数据有一些差异，（比如*2671-> 13293**，**2672 -> 13294*）当前还没有查出来什么地方修改的，如何修改的，后续会继续研究。

