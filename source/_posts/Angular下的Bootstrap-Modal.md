---
title: Angular下的Bootstrap Modal
date: 2019-02-18 16:27:10
tags:
    - Angular
    - Typescript
    - 前端
---
# 前言
Bootstrap有一个Angular的版本——ngx-bootstrap，里面是有提供Boostrap Modal的，但是我一直觉得这种Modal的形式不太好。因为Modal本来就是相对独立在页面之外的，如果要把Modal的代码也写到当前页面里的话其实反而破坏了页面本身的结构。所以我自己封装了一个Modal，写这篇博客记录一下封装的过程。

# Modal Component
不管Modal的html部分写在什么地方，Component都是必需的。
```html
  <div (click)="onContainerClicked($event)" class="modal fade" tabindex="-1" [ngClass]="{'in': visibleAnimate}"
       [ngStyle]="{'display': visible ? 'block' : 'none', 'opacity': visibleAnimate ? 1 : 0}">
    <div class="modal-dialog">
      <div class="modal-content">
        <ng-template modal-host></ng-template>
      </div>
    </div>
  </div>
```
template的部分非常简单。因为我希望Modal的主体是可以自定义的，所以就在里面使用了`ng-template`标签。
```Typescript
@Directive({
  selector: '[modal-host]',
})
export class ModalDirective {
  constructor(public viewContainerRef: ViewContainerRef) { }
}
```
modal-host对应的就是这段代码。  
接下来就是Modal Component的主体部分了
```Typescript
export class ModalComponent{

  public visible = false;
  public visibleAnimate = false;
  @ViewChild(ModalDirective) modalHost:ModalDirective;

  constructor(private componentFactoryResolver: ComponentFactoryResolver,private modal:ModalService){
    this.modal.show.subscribe(value => {
      this.show(value);
    })
  }

  public show(item:ShowItem): void {
    //用工厂从component类型里生成component
    let componentFactory = this.componentFactoryResolver.resolveComponentFactory(item.component);
    let viewContainerRef = this.modalHost.viewContainerRef;
    viewContainerRef.clear();
    let componentRef = viewContainerRef.createComponent(componentFactory);
    (<ModalValue>componentRef.instance).callback = item.callback;
    (<ModalValue>componentRef.instance).params = item.params;
    let modal = this;
    (<ModalValue>componentRef.instance).close = ()=>{modal.hide()};
    (<ModalValue>componentRef.instance).onInit();
    this.visible = true;
    setTimeout(() => this.visibleAnimate = true, 100);
  }

  public hide(): void {
    this.visibleAnimate = false;
    setTimeout(() => this.visible = false, 300);
  }

  public onContainerClicked(event: MouseEvent): void {
    if ((<HTMLElement>event.target).classList.contains('modal')) {
      this.hide();
    }
  }
}

export interface ModalValue {
  callback:(any)=>void;
  close:()=>void;
  params:any;
  onInit();
}

export class ShowItem{
  component:Type<ModalValue>;
  callback:(any)=>void;
  params:any;
  constructor(item:Type<any>,params:any,callback:(any)=>void){
    this.component = item;
    this.callback = callback;
    this.params = params;
  }
}

```
关键部分在`show`方法里。当从Modal Service(后面会写)里拿到`item`之后就会调用`show`方法。首先是拿到新的component，清除之前的component，接着把`params`，`callback`注入到component中，调用`onInit`方法。最后显示Modal。

Component写完之后需要在app.component.html里给它腾个位置，不然没地方显示。

# Modal Service
```Typescript
@Injectable({
  providedIn: 'root'
})
export class ModalService {
  show: Observable<ShowItem>;
  private showIn: Subject<ShowItem>;
  constructor() {
    this.showIn = new Subject<ShowItem>();
    this.show = this.showIn.asObservable();
  }

  modal(component:Type<any>,params:any):Promise<any>{
    return new Promise((resolve) => {
      this.showIn.next(new ShowItem(component, params, resolve));
    })
  }
}
```
Modal Service部分就非常简单了。创建一个`Obervable`给Modal Component订阅。每当其他地方调用`modal`方法的时候就会把component类型和参数发布出去。

# 使用举例
以一个非常简单的登出提示框为例
```Typescript
@Component({
  template: `
    <div class="modal-header">
      登出确认
    </div>
    <div class="modal-body">
      <p>是否要退出当前账号</p>
    </div>
    <div class="modal-footer">
      <button class="btn btn-primary" (click)="finish()">确定</button>
    </div>
  `
})
export class LogoutEnsure implements ModalValue {
  @Input() callback: (any) => void;
  @Input() close: () => void;
  @Input() params: any;

  onInit() {
  }

  finish() {
    this.callback({});
    this.close();
  }
}
....
  logout() {
    this.modal.modal(LogoutEnsure, {}).then(() => {
      this.api.getLogout().subscribe(()=>{
        this.router.navigate(['/']);
      });
    });
  }
....
```
Component只要继承`ModalValue`就可以在Modal中显示出来。调用modal的时候只要传Component类型即可。

# 小结
自己实现一个Modal还是非常简单的。这样可以避免像ngx-bootstrap一样dom节点过深，而且代码结构也更加美观。