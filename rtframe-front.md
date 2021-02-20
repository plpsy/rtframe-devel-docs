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
后面的逻辑不变

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

step增加一个资源需求指数的字段,默认值是1
"res_level":1

