工具使用参考  https://github.com/medcl/elasticsearch-migration

工具下载      wget https://github.com/medcl/esm/releases/download/v0.4.1/linux64.tar.gz

解压后，进入底层目录:	

	cd bin/linux64/

每个Index的迁移分为三项操作：

1. 创建index
2. 创建mapping
3. 使用esm迁移数据

具体命令如下：

	curl -XPUT https://xxx.com/new_launcher_item/
	curl -H "Content-Type: application/json" -XPUT https://xxx.com/new_launcher_item/_mapping/default -d '{"default":{"properties":{"contents":{"type":"nested","properties":{"description":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"langCode":{"type":"keyword"},"title":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}}}},"createTime":{"type":"long"},"itemOnUse":{"type":"long"},"itemType":{"type":"long"},"itemUuid":{"type":"keyword"},"restrictions":{"type":"keyword"},"sourceUuid":{"type":"keyword"},"updateTime":{"type":"long"}}}}'
	./esm  -s http://es:9200   -d https://xxx.com -x new_launcher_item -w=5 -b=10 -c 50

	curl -XPUT https://xxx.com/recommend_cell_program/
	curl -H "Content-Type: application/json" -XPUT https://xxx.com/recommend_cell_program/_mapping/program -d '{"program":{"properties":{"areaCodeThree":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"areaCodeTwo":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"channelUuids":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"contentType":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"contentUuid":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"id":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"langContents":{"properties":{"codeThree":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"codeTwo":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"fieldName":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"fieldValue":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}}}},"releaseTime":{"type":"long"},"sort":{"type":"long"},"sourceUuid":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"thirdUuid":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"thumbnails":{"properties":{"format":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"height":{"type":"long"},"rate":{"type":"float"},"url":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"width":{"type":"long"}}}}}}'
	./esm  -s http://es:9200   -d https://xxx.com -x recommend_cell_program -w=5 -b=10 -c 50

