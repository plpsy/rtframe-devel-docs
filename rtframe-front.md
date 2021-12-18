# rtFrame前端开发

## 物理资源的json文件格式

* 利用文件上传下载功能实现物理资源的配置文件修改

获取 GET /repapi/file/config/rtframe-hubcfg.json

修改 POST /repapi/file/config/

* 格式说明

通过内置tag支持芯片类型、芯片型号, 如chipType:ppc,chipModel:2020
rack是板卡名称与插槽号拼接而成, 如SCM_1-(SLOT1)

```
{
 "190": {
    "dataCenter": "机箱2",
    "rack": "NPM1",
    "tags": [
      "FPGA-ON:NPM",
      "RIOID:190"      
    ]
	},
	"16": {
    "dataCenter": "机箱1",
    "rack": "CNIP2_1",
    "tags": [
      "FPGA-ON:CNIP",
      "RIOID:16"      
    ]
	},
	"158": {
    "dataCenter": "机箱1",
    "rack": "CNIP3_0",
    "tags": [
      "DSP-ON:CNIP",
      "RIOID:158"      
    ]
	},	
	"159": {
    "dataCenter": "机箱1",
    "rack": "CNIP3_1",
    "tags": [
      "DSP-ON:CNIP",
      "RIOID:159"
    ]
	},	
	"162": {
    "dataCenter": "机箱1",
    "rack": "CNIP3_0",
    "tags": [
      "FPGA-ON:CNIP",
      "RIOID:162"      
    ]
	},
	"164": {
    "dataCenter": "机箱1",
    "rack": "CNIP3_1",
    "tags": [
      "FPGA-ON:CNIP",
      "RIOID:164"      
    ]
	}
    ...	
}
```

## 任务编排的json文件格式

* json文件demo
```
{
	"dataSource": [{
		"args": ["fpga_transfer:1.0"],
		"name": "dataSource",
		"partition_count": 1,
		"path": "fpga_transfer:1.0",
		"tags": [{
			"key": "RIOID",
			"value": "185"
		}],
		"type": "fpga"
	}],
	"dataset_manager_on": false,
	"description": "FPGA圆周64核",
	"dsp_instruction_map": [],
	"executables": ["org:1.4", "fpga_transfer:1.0"],
	"fpga_instruction_map": [],
	"mtime": 1613807174435,
	"name": "Sar",
	"restart_on_error": false,
	"steps": [{
		"input": {
			"ref": ["dataSource"],
			"type": "dataSource"
		},
		"name": "op0",
		"ops": [{
			"args": ["src", "org:1.4", "srcMap"],
			"op": "Map",
			"processor_type": "dsp"
		}],
		"tags": [{
			"key": "DSP-ON",
			"value": "NPM"
		}, {
			"key": "RIOID",
			"value": "54"
		}]
	}, {
		"input": {
			"ref": ["op0"],
			"type": "step"
		},
		"name": "op1",
		"ops": [{
			"args": ["roundRobin", "org:1.4", "8"],
			"op": "RoundRobin",
			"processor_type": "dsp"
		}],
		"tags": [{
			"key": "DSP-ON",
			"value": "NPM"
		}]
	}, {
		"input": {
			"ref": ["op1"],
			"type": "step"
		},
		"name": "op2",
		"ops": [{
			"args": ["roundRobin", "org:1.4", "8"],
			"op": "RoundRobin",
			"processor_type": "dsp"
		}],
		"tags": [{
			"key": "DSP-ON",
			"value": "NPM"
		}]
	}, {
		"input": {
			"ref": ["op2"],
			"type": "step"
		},
		"name": "op3",
		"ops": [{
			"args": ["Map", "org:1.4", "mySarMap"],
			"op": "Map",
			"processor_type": "dsp"
		}],
		"tags": [{
			"key": "DSP-ON",
			"value": "NPM"
		}]
	}, {
		"input": {
			"ref": ["op3"],
			"type": "step"
		},
		"name": "op4",
		"ops": [{
			"args": ["MergeTo", "org:1.4", "8"],
			"op": "MergeTo",
			"processor_type": "dsp"
		}],
		"tags": [{
			"key": "DSP-ON",
			"value": "NPM"
		}]
	}, {
		"input": {
			"ref": ["op4"],
			"type": "step"
		},
		"name": "op5",
		"ops": [{
			"args": ["Map", "org:1.4", "mySarmap1"],
			"op": "Map",
			"processor_type": "dsp"
		}],
		"tags": [{
			"key": "DSP-ON",
			"value": "NPM"
		}]
	}, {
		"input": {
			"ref": ["op5"],
			"type": "step"
		},
		"name": "op6",
		"ops": [{
			"args": ["MergeTo", "org:1.4", "1"],
			"op": "MergeTo",
			"processor_type": "dsp"
		}],
		"tags": [{
			"key": "DSP-ON",
			"value": "NPM"
		}]
	}, {
		"input": {
			"ref": ["op6"],
			"type": "step"
		},
		"name": "op7",
		"ops": [{
			"args": ["output", "org:1.4"],
			"op": "DspOutput",
			"processor_type": "dsp"
		}],
		"tags": [{
			"key": "DSP-ON",
			"value": "NPM"
		}, {
			"key": "RIOID",
			"value": "55"
		}]
	}],
	"version": "29.0"
}
```
* 综合化模式下的简化

取消dataSource字段

step列表中，如果是第一个step,则固定设置
```
"input": 
{
    "ref": ["null"],
    "type": "null"
},
```

**后面的step的通过input字段，引用前一个组件，形成一个单向链表，如果一个组件被删除后，需要修改input字段形成新的链表**
```
"input": 
{
    "ref": ["前一个step名称"],
    "type": "step"
},
```

组件组就是一个step，组件组下的组件就是ops，包含一个或多个，每个op格式
```
{
    "args": ["组件名称", "算法文件名称", "null"],
    "op": "Map",
    "processor_type": "计算类型"
}
```
如
```
{
    "args": ["uv", "org:1.4", "null"],
    "op": "Map",
    "processor_type": "dsp"
}
```

op增加一个资源需求指数的字段,默认值是1
```
{
    "args": ["uv", "org:1.4", "null"],
    "op": "Map",
    "processor_type": "dsp",
	"res_level" : 1
}
```

## 新增rest api

* 获取机箱列表概要信息

GET /api/res

* 获取指定机箱下的板卡列表概要信息

GET /api/res/板卡名称

* 获取指定板卡下的芯片列表信息

GET /api/res/机箱名称/板卡名称

* 获取机箱列表详细信息,详细信息至芯片

GET /api/resources

* 获取任务的综合化应用视图信息

GET /api/comjobs/任务ID

* 日志获取接口

GET /api/logs/芯片id/计算核id

* 算法参数注入接口

POST /api/algparam/芯片id/计算核id

通过Multipart/form-data POST的方式注入算法参数,上传的文件标识是paramfile

通过curl指令操作的例子如下:

curl -i -X POST -F "paramfile=@./param" http://www.rtframe.cn/api/algparam/50/1

* 算法重构接口

PUT /api/runningalg/芯片id/计算核id/算法名称/算法版本

算法名称与算法版本,是指软件算法仓库中的算法与版本,用户通过搜索选择下发要重构的算法。

通过curl指令操作的例子如下:

curl -i -X PUT http://www.rtframe.cn/api/runningalg/50/1/number/1.0

* 板卡参数配置

POST /api/boardCfg/芯片id/计算核id

通过Multipart/form-data POST的方式注入配置文件,上传的文件标识是paramfile

通过curl指令操作的例子如下:

curl -i -X POST -F "paramfile=@./boardcfg.json" http://10.10.10.126/api/boardCfg/1/0

* 板卡健康信息

POST /api/boardHealthCfg/芯片id/计算核id

通过Multipart/form-data POST的方式注入配置文件,上传的文件标识是paramfile

通过curl指令操作的例子如下:

curl -i -X POST -F "paramfile=@./healthCfg.json" http://10.10.10.126/api/boardHealthCfg/1/0

* master集群ip配置

POST /api/masterIpCfg/芯片id/计算核id

通过Multipart/form-data POST的方式注入配置文件,上传的文件标识是paramfile

通过curl指令操作的例子如下:

curl -i -X POST -F "paramfile=@./connectIP.json" http://10.10.10.126/api/masterIpCfg/1/0

* zynq基础程序远程烧写

POST /api/zynqImg/芯片id/计算核id

通过Multipart/form-data POST的方式注入配置文件,上传的文件标识是paramfile

通过curl指令操作的例子如下:

curl -i -X POST -F "paramfile=@./zynq.bin" http://10.10.10.126/api/zynqImg/1/0



* 频综b码相关

curl -I -X POST www.rtframe.cn:/api/freqMB/box1/0

curl -i -X POST www.rtframe.cn:/api/bcodeMB/box1/1

curl -i -X POST www.rtframe.cn:/api/bcodeIO/box1/2

curl -i -X POST www.rtframe.cn:/api/spSlotFreq/box1/1/1

curl -i -X POST www.rtframe.cn:/api/dpSlotFreq/box1/1/1

前3个命令需要芯片包含”zynq“标签    后两个命令需要 ”zynq“ ”infoSwitch“标签

* 指定部署 ：

JSON 格式：<br>
type JsonSpLocation struct { <br>
	Affinity string                 `json:"affinity"` <br>
	StepTags []stepTags             `json:"stepTags"` <br>
}<br>
type stepTags struct {<br>
	Name string                     `json:"name"` <br>
	Tags AgentTag                   `json:"tags"`<br>
}<br>
type AgentTag struct {<br>
	Key         string              `json:"key"` <br>
	Value       string              `json:"value"` <br>
}<br>


curl  -i  -H "Content-Type: application/json" --data "" -X POST 127.0.0.1/api/jobs/T_LDT/1.0<br>

curl  -i  -H "Content-Type: application/json" --data <br>
"{\"dc\":\"35\",\"stepTags\":[{\"name\":\"jdc_1\",\"tags\":[{\"key\":\"zynq\",\"value\":\"0\"},{\"key\": \"infoSwitch\",\"value\":\"0\"}]},<br>{\"name\":\"jdc_2\",\"tags\":[{\"key\":\"infoSwitch\",\"value\":\"0\"},{\"key\":\"zynq\",\"value\":\"0\"}]}]}" -X POST 127.0.0.1/api/jobs/T_LDT/1.0


* 指定芯片添加标签：POST /api/AddAgentTags/芯片id/KEY/VAL

curl -i -X POST www.rtframe.cn/api/AddAgentTags/202/FPGA/1


* 指定芯片删除标签：DELETE /api/AddAgentTags/芯片id/KEY/VAL

curl -i -X DELETE www.rtframe.cn/api/AddAgentTags/202/FPGA/1


* 修改芯片状态 ：PUT api/chipStatus/id/status

 curl -i -X PUT 127.0.0.1/api/chipStatus/201/maintance

* 修改板卡状态 ：PUT api/chipStatus/dc/rack/status

curl -i -X PUT 127.0.0.1/api/boardStatus/box1/demo%7C1/active


* 板卡健康  GET api/clientHealth/dc/rack

curl -i 127.0.0.1/api/clientHealth/box1/demo%7C1


* 机箱  GET api/clientHealth/dc/

curl -i 127.0.0.1/api/clientHealth/box1

