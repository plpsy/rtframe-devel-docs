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
