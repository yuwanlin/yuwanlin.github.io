---
title: React render是天然的模板方法
date: 2020-01-01
tags:
- React
- 设计模式
categories:
- 工作
---

## 定义
模板方法模式中，抽象父类定义一个模板方法，模板方法中封装了程序的步骤或者框架，其中有一些共同的步骤父类提供默认的实现。对于不同的部分，父类提供抽象方法，不同的子类根据场景需要重写父类抽象方法。父类模板方法应当是不能被重写的，对应在java中就是final关键字，但是js中没有这个关键字。

## 适用场景
模板方法模式适用于存在多个相同场景/步骤的情况下。比如泡茶和泡咖啡，都需要煮开水、放入原料、搅拌等步骤。虽然步骤名称相同，但是具体处理的内容可能是不同的（泡茶放入茶叶，泡咖啡放入咖啡粉末）。

在前端，适用了那些同一个场景下多个不同的子场景，子场景拥有相同的元素（如图片，按钮，编辑框等），但是样式或者具体逻辑有所差别。


## 自定义模板方法
在使用`react`开发前端的过程中，一直没发现有什么好的方法使用模板方法模式，因为react的组件都是继承`React.Component`的，然后组件内实现render方法来输出内容。而导出的组件通过JSX写法`(<Component />)`这种方式使用。对于开发者来说，它没有涉及到实例化这个步骤。强行使用模板方法模式会显得有些奇怪。

![](/assets/render-template/images/class-pic.png)


上图中，`renderStudyInfo`是模板方法，模板方法中调用两个方法，用于渲染UI。其中一个是抽象方法，用于让子类去实现。另一个方法父类中定义了默认的实现，子类也可以重写该方法。

```js
private createStudyInfoStrategy(data: ModuleData) {
  const { source } = this.props;
  let studyContent: StudyContent;
  const { courseType, trainSubType } = data;
  switch (courseType) {
    case COURSE_TYPE.CLASSICAL_COURSE:
      studyContent = new SpecialCourse({ moduleData: data });
      break;
    case COURSE_TYPE.TRAINING_CAMP:
      if(trainSubType === TrainSubType.TrainingCamp) {
        studyContent = new TrainingCampV2({ moduleData: data })
      } else if(trainSubType === TrainSubType.GeneralClass || trainSubType === TrainSubType.RequiredClass) {
        studyContent = new GeneralAndRequiredClass({ moduleData: data, source })
      } else {
        studyContent = new OtherContent({ moduleData: data });
      }
      break;
    default:
      studyContent = new OtherContent({ moduleData: data });
      break;
  }
  return studyContent;
}

renderCourseInfo(data = {} as ModuleData) {
  return this.createStudyInfoStrategy(data).renderStudyInfo();
}
```

使用过程中，`createStudyInfoStrategy`相当于一个简单工厂方法，通过多态，可以定义实例的类型是`StudyContent`类型。得到具体的实例后，调用模板方法`renderStudyInfo`。

为什么说奇怪呢，就在于子类都是继承父类`StudyContent`的，而父类又继承自`React.Component`。但是在使用子类的时候，我们又实例化了这个子类，第一个参数是React组件的props，通过这种方式子类可以使用props内容。

## render是天然的模板方法
后来又做了一个需求，觉得实例化子类的方式显得不太优雅。就想到能否利用render方法作为模板方法呢。这样利用React的特性，它的内部会调用render方法，就不需要开发者再手动实例化子类，然后去调用模板方法了。事实证明是可行的。父类代码如下。
```js
abstract class SendSuccessTemplate extends React.Component<SendSuccessTemplateProps, {}> {

  abstract renderSubjectText(): JSX.Element;
  abstract renderDefaultImage(): JSX.Element;
  abstract renderUserText(): JSX.Element;
  abstract renderDate(): JSX.Element;
  abstract renderUserName(): JSX.Element;

  renderQrCode(style: React.CSSProperties = {}) {
    //
  }

  renderText() {
    return (
      <>
        { this.renderSubjectText() }
        { this.renderUserText() }
      </>
    )
  }

  render() {
    const { currTemplate, hasQrCode } = this.props;
    return (
      <div styleName='wrapper'>
        <div styleName={`send-success-template${currTemplate}bg ${hasQrCode ? 'hasQrCode' : ''}`}>
          { this.renderDefaultImage() }
          { this.renderText() }
          { this.renderQrCode() }
          { this.renderUserName() }
          { this.renderDate() }
        </div>
      </div>
    )
  }
}
```

由于将render作为模板方法，那么子类继承父类后，只需要实现抽象方法即可，而子类不用再实现render方法了。使用的时候直接把子类当做React组件使用。就不需要再向上面的方式一样，实例化后再调用对应的模板方法了。使用方式如下：
```js
function CreateSendSuccessTemplate() {
  const { currTemplate } = window.data;
  switch(currTemplate) {
    case Template.One:
      return <SendSuccessTemplate1 { ...window.data } />
    case Template.Two:
      return <SendSuccessTemplate2 { ...window.data } />
    case Template.Three:
      return <SendSuccessTemplate3 { ...window.data } />
    case Template.Four:
      return <SendSuccessTemplate4 { ...window.data } />
    case Template.Feedback:
      return <FeedbackCard { ...(window as any).data } />
    default:
      throw new Error('未知的模板类型');
  }
}
```

## 总结
前端由于其特殊性，不容易实现模板方法。将render作为模板方法是一个尝试，事实证明是可行的。

### 优点
1. 使用模板方法模式来组织代码，代码可以得到极大程度的复用。
2. 在将来如果需要添加新的场景，只需要添加一个新的模板子类即可。

### 缺点
1. 模板方法不能被子类重写，但是ts并没有提供这一机制来保证子类不能重写父类方法，这一点需要开发人员明确。
2. 在使用了`react-css-modules`的情况下，如果公共方法是写在父类的，那么对应的样式文件一定要包括在父类的样式文件中。不过子类可以通过传入style的方法使用内联样式。
