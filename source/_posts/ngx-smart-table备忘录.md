---
title: ngx-smart-table备忘录
date: 2019-10-28 18:25:45
tags:
    - Angular
    - 备忘录
---

# 基本信息
官网
> https://akveo.github.io/ng2-smart-table/

github
> https://github.com/akveo/ng2-smart-table

examples
> https://github.com/akveo/ng2-smart-table/tree/master/projects/demo/src/app/pages

## 基本用法
```typescript
@Component({
  selector: 'ngx-data',
  template: `<ng2-smart-table [settings]="settings" [source]="source">`,
})
export class DataComponent {
  option:any;
  settings = {
    pager: {
      display: true,
      perPage: 50
    },
    actions: {  //修改ng2-smart-table自带的action列
      add: false,  
      edit: false,
      delete: false,
    },
    hideSubHeader: true, //隐藏顶栏
    columns: {
      col1: {
        title: '值',
        type: 'string',
      },
      col2: {
        title: '更新时间',
        type: 'string',
      },
    },
  };
  source: LocalDataSource;  
}
```

# 分页器

```javascript
pager:{
    display: true , //是否显示分页器 
    perPage: 50 //每页显示数量
}
```
## 后端分页

后端分页可以继承`LocalDataSource`,写一个新的`DataSource`，重写`getElements()`和`count()`函数
```typescript
export class MyDataSource extends LocalDataSource {
  all: number;
  devName: string;
  namFilter:string;
  constructor(private api: MyService) {
    super()
  }

  count(): number {
    return this.all;
  }

  getElements(): Promise<any> {
    return this.api.getData(nameFilter).pipe(map(result=>{  //假设返回为{'count':100,data:[.......]}
        this.all = result.count;
        return result.data;
    })).toPromise();
  }
}
```
使用方式
```typescript
source:MyDataSource = new DeviceDataSource(this.api);

search(name:string){
    this.source.nameFilter = name;
    this.source.setPage(1);
    this.source.refresh();
}
```

# 自定义渲染

## 方案1
```javascript
columns: {
      col1: {
        title: '链接',
        type: 'html',
      },
}
```
直接修改值为html代码

## 方案2 自定义渲染器
```typescript
import {Component, Input, OnInit} from "@angular/core";
import {ViewCell} from "ng2-smart-table";

@Component({
  template: `
    <button (click)="onClick()">{{renderValue}}</button>
  `,
})
export class CustomRender implements ViewCell, OnInit {
  renderValue: string;

  @Input() value: string | number; //对应列的值
  @Input() rowData: any; //本行所有值:{'col1':'value1','col2','value2'}

  ngOnInit() {
    this.renderValue = value;
  }

  onClick(){
      console.log(rowData);
  }
}
```
```typescript
columns: {
      col1: {
        title: '链接',
        type: 'custom',
        renderComponent: CustomRender
      },
}
```