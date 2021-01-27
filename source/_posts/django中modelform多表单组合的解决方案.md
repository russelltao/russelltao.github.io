---
title: django中ModelForm多表单组合的解决方案
tags:
  - django
  - form
id: '53'
categories:
  - - web
date: 2017-01-25 10:41:07
---

django是[Python](http://lib.csdn.net/base/python "Python知识库")语言快速实现web服务的大杀器，其开发效率可以非常的高！但因为秉承了语言的灵活性，django框架又太灵活，以至于想实现任何功能都有种“条条大路通罗马”的感觉。这么多种选择放在一起，如何分出高下？我想此时的场景下就两个标准： 1、相同的功能用最少的代码实现（代码少BUG也会少）； 2、相对最易于理解，从而易于维护和扩展。 书归正传，web服务允许用户输入，基本上要靠表单。而django对表单的支持力度非常大，我们用不着在浏览器端的html文件里写大量<form>代码，再到web端去匹配form里的id/name/value、验证规则，再与持久层[数据库](http://lib.csdn.net/base/mysql "MySQL知识库")比较并做操作。我们需要完成的工作非常少，可以没有相似的重复代码。有些复杂的场景，会要求一个表单的内容存放到多张表里，本文将通过4个部分，阐述它的实现方法。 1、django基础表单的功能 定义一个表单非常简单，继承类django.forms.Form即可，例如：

```
class ProjectForm(forms.Form):  
    name = forms.CharField(label='项目名称', max_length=20)  
```

这个表单类可以生成HTML形式的form，可以从request.POST中解析form到ProjectForm类实例。怎么做到的呢？ 看下django.forms.Form定义：

```
class Form(six.with_metaclass(DeclarativeFieldsMetaclass, BaseForm)):  
    "A collection of Fields, plus their associated data."  
    # This is a separate class from BaseForm in order to abstract the way  
    # self.fields is specified. This class (Form) is the one that does the  
    # fancy metaclass stuff purely for the semantic sugar -- it allows one  
    # to define a form using declarative syntax.  
    # BaseForm itself has no way of designating self.fields.  
```

注释说得很清楚，Form这个类就是为了实现declarative syntax的，也就是说，继承了Form后，我们直观的表达ProjectForm里要有一个Field名叫name，不关心其语法实现，而通过Form多继承中的DeclarativeFieldsMetaclass语法糖，将会把name弄到类实例的self.fields里。

我们重点关注表单的BaseForm类，它实现了基本的逻辑。截选了一小段对接下来的陈述有意义的代码，做一个简单的注释。

```
class BaseForm(object):  
    def __init__(self, data=None, files=None, auto_id='id_%s', prefix=None,  
                 initial=None, error_class=ErrorList, label_suffix=None,  
                 empty_permitted=False, field_order=None, use_required_attribute=None):  
        #data参数用于接收request.POST字典，如果是GET方法就不传  
    self.data = data or {}  
    #files用于接收request.FILES，也就是处理上传文件  
    self.files = files or {}  
    #本篇文章的重点在于多个表单集成到一个form中，此时为防止有同名的field，需要加prefix前缀  
        if prefix is not None:  
            self.prefix = prefix  
    #GET显示表单时，如果要显示初始值，请用initial参数  
        self.initial = initial or {}  
  
    #模板中显示{{form}}时，默认是以<table></table>显示的  
    def __str__(self):  
        return self.as_table()  
  
    #如果模板中不想写重复代码，只以固定的格式来显示每一个field，那么就用{% for field, val in form %}来遍历处理吧  
    def __iter__(self):  
        for name in self.fields:  
            yield self[name]  
  
    #如果传入了prefix参数，html中每个field的name和id里都会加上prefix前缀  
    def add_prefix(self, field_name):  
        return '%s-%s' % (self.prefix, field_name) if self.prefix else field_name  
  
    #模板中以html格式显示form就靠这个方法  
    def _html_output(self, normal_row, error_row, row_ender, help_text_html, errors_on_separate_row):  
        "Helper function for outputting HTML. Used by as_table(), as_ul(), as_p()."  
        top_errors = self.non_field_errors()  # Errors that should be displayed above all fields.  
        output, hidden_fields = [], []  
  
    #除了默认的table方式显示外，还可以<ul><li>或者<p>方式显示  
    def as_table(self):  
        "Returns this form rendered as HTML <tr>s -- excluding the <table></table>."  
    def as_ul(self):  
        "Returns this form rendered as HTML <li>s -- excluding the <ul></ul>."  
    def as_p(self):  
        "Returns this form rendered as HTML <p>s."  
```

所以，基本表单的功能看BaseForm已经足够了。

2、从模型创建表单 django对于MVC中的C与M间的映射是非常体贴的，集中体现中Model模型中（比如模型的权限与用户认证）。那么，一个模型代表着RDS中的一张表，模型的实例代表着关系数据库中的一行，而form如何与一行相对应呢？ 定义一个模型引申出的表单非常简单，例如：

```
class ProjectForm(ModelForm):  
    class Meta:  
        model = Project  
        fields = ['approvals','manager','name','fund_rource','content','range',]  
```

在model中告诉django模型是谁，在fields中告诉django需要在表单中创建哪些字段。django会有一个django.db.models.Field到django.forms.Field的转换规则，此时会生成Form。我们看看ModelForm是什么样的：

```
class ModelForm(six.with_metaclass(ModelFormMetaclass, BaseModelForm)):  
    pass  
```

类似Form类，ModelFormMetaclass就是语法糖，我们重点看BaseModelForm类：

```
class BaseModelForm(BaseForm):  
    def __init__(self, data=None, files=None, auto_id='id_%s', prefix=None,  
                 initial=None, error_class=ErrorList, label_suffix=None,  
                 empty_permitted=False, instance=None, use_required_attribute=None):  
        opts = self._meta  
    #相比较BaseForm，多了instance参数，它等价于Model模型的一个实例  
        if instance is None:  
            #不传instance参数，则会新构造model对象  
            self.instance = opts.model()  
            object_data = {}  
        else:  
            self.instance = instance  
            object_data = model_to_dict(instance, opts.fields, opts.exclude)  
    #此时传递了initial也一样可以生效，同时还会设置到Model中  
        if initial is not None:  
            object_data.update(initial)  
  
    def save(self, commit=True):  
    #默认commit是True，此时就会保存Model实例到数据库  
        if commit:  
            self.instance.save()  
    #同时保存many-to-many字段对应的关系表  
            self._save_m2m()  
        else:  
    #注意，本篇文章主要用到commit=False这个参数，它会返回Model实例，允许我们在修改instance后，在instance上再调用save方法  
            self.save_m2m = self._save_m2m  
        return self.instance  
```

所以，对于ModelForm我们可以传入instance参数初始化表单，可以调用save()方法直接将从html里得到的表单数据持久化到数据库中。而我们只需要几十行代码就可以完成这么多工作。 3、通用视图 django.views.generic.ListView和django.views.generic.edit下的CreateView, UpdateView, DeleteView都是通用视图。即，我们又可以通过它们，把很多重复的工作交给django完成，又可以少写很多代码完成同样的功能了。这里仅以CreateView为例说明，因为它相对最复杂，接下来的多ModelForm的提交也是在CreateView上进行的。 通用视图使用时，只需要承继后，再设置model或者form\_class即可。比如CreateView就会由django自动的把页面上POST出的form数据解析到model生成的表单（或者form\_calss指定的ModelForm类型表单），同时调用表单的save方法将数据添加到模型对应的数据库表中。当然GET请求时会生成空form到页面上。可以看到，除去定义model或者form类外，几行代码就可以搞定这么多事。我们看看CreateView的继承关系:  简单介绍下CreateView通用视图中每个父类的作用。

1.  View是所有视图类的父类，根据方法名分发请求到具体的get或者post等方法，提供as\_view方法。
2.  TemplateResponseMixin提供render\_to\_response方法将响应通过context上下文在模板上渲染。
3.  ContextMixin在context上下文中加入'view'元素，值为self实例。
4.  ProcessFormView在GET请求上渲染表单，在POST请求上解析form到表单实例。注意，它会在post请求中判断表单是否可用，is\_valid为真时，会调用form\_valid方法，因此，重写form\_valid方法是第4部分处理多model到一个form的关键。
5.  FormMixin允许处理表单，可指定form\_class为某个表单。
6.  SingleObjectMixin生成context上下文，同时根据model模型名称生成object并添加到上下文中的'object'元素。
7.  ModelFormMixin提供在请求中处理modelform的方式。
8.  SingleObjectTemplateResponseMixin帮助TemplateResponseMixin提供模板。

所以，在用CreateView、一个模型、一个模板实现添加一行记录的功能时是多么简单，因为这些父类会自动生成object，渲染到模板，解析form表单，save到数据库中。所以，从模型创建出的表单ModelForm，配合上通用视图后，威力巨大！！ 4、多个ModelForm在一个form里提交 终于可以回到本文的主题了。CreateView默认是处理一个Model模型、一个ModelForm表单的，然而，很多时候为了解耦，会把一张表拆成多张表，通过id关联在一起。在django的模型中就体现为ForeignKey、ManyToManyField或者OneToOneField。而在业务逻辑上，需要体现为一张表单，对应着数据库里的多张表。 例如，我们希望录入合同，其中合同Model中还有地址Model和项目Model，而项目Model中又有地址Model，等等。 当然，我们有很多种实现的方案，但是，前面三部分说了那么多，不是浪费口水的。我们已经有了通用视图+ModelForm这样的利器，难道还需要手动去写Form表单？我们已经习惯了在Model里定义好类型和有点注释作用还能当label的verbose\_name，还需要在forms.Form里再来一遍？还需要在视图中写这么通用的逻辑代码吗？当然不用。 inlineformset\_factory是一种方案，但它限制太多，而且有些晦涩，我个人感觉是不太好用的。 那么，从第1部分我介绍的Form里的prefix，以及第3部分里类图中的ProcessFormView允许重定义form\_valid，以及第2部分中ModelForm的save方法的行为控制，解决方案已经一目了然了。 拿上面提到的例子来说，我们创建合同时，指明了项目，包括项目地址和合同签订地址，这涉及到三张表和四条记录（地址表有两条）。 我们三张表的模型如下：

```
class PrimeContract(models.Model):  
    address = models.ForeignKey(Address, related_name="prime_contract_address", verbose_name="address")  
    project = models.ForeignKey(Project, related_name="prime_contract", verbose_name="project")  
class Project(models.Model):  
    address = models.ForeignKey(Address, related_name="project_address", verbose_name="project address")  
class Address(models.Model):  
    pass  
```

接着，定义ModelForm表单，这非常简单：

 

```
class AddressForm(ModelForm):  
    class Meta:  
        model = Address  
        fields = ...  
class ProjectForm(ModelForm):  
    class Meta:  
        model = Project  
        fields = ...  
class PrimeContractForm(ModelForm):  
    class Meta:  
        model = PrimeContract  
        fields = ...  
```

再写视图，这里要重写2个方法：

```
class PrimeContractAdd(CreateView):  
    success_url = ...  
    template_name = ...  
    form_class = PrimeContractForm  
    def get_context_data(self, **kwargs):  
        context = super(PrimeContractAdd, self).get_context_data(**kwargs)  
        #SingleObjectMixin父类只会处理PrimeContractForm表单，另外三条数据库记录对应的表单我们要自己处理了，此时prefix派上用场了，因为Field重名是百分百的事  
        if self.request.method == 'POST':  
            contractAddressForm = AddressForm(self.request.POST, prefix='contractAddressForm')  
            projectAddressForm = AddressForm(self.request.POST, prefix='projectAddressForm')  
            projectForm = ProjectForm(self.request.POST, prefix='projectForm')  
        else:  
            contractAddressForm = AddressForm(prefix='contractAddressForm')  
            projectAddressForm = AddressForm(prefix='projectAddressForm')  
            projectForm = ProjectForm(prefix='projectForm')  
        #注意要把自己处理的表单放到context上下文中，供模板文件使用  
        context['contractAddressForm'] = contractAddressForm  
        context['projectAddressForm'] = projectAddressForm  
        context['projectForm'] = projectForm  
        return context  
      
    #重写form_valid，父类ProcessFormView会在PrimeContractForm表单is_valid方法返回True时调用该方法  
    def form_valid(self, form):  
        #首先我们要获取到PrimeContractForm表单对应的模型，此时是不能save的，因为外键project和address对应的数据库记录还没有创建，所以commit传为False  
        contract = form.save(commit=False)  
        #获取上面get_context_data方法中在POST里得到的表单  
        context = self.get_context_data()  
        #按照四条数据库记录的顺序依次的创建（调用save方法）、主键赋到下一条记录的外键中、下一次记录创建（save）  
        projectAddress = context['projectAddressForm'].save()  
        #从项目表单中获取到模型，先把地址的id赋到外键上再保存  
        project = context['projectForm'].save(commit=False)  
        project.address = projectAddress  
        project.save()  
        contractAddress = context['contractAddressForm'].save()  
        #将合同模型中的address和project都设置好后再保存  
        contract.address = contractAddress  
        contract.project = project  
        contract.save()  
          
        return super(PrimeContractAdd, self).form_valid(form)  
```

最后写模板：

```
#这三个表单我们手动处理过的  
{{ contractAddressForm }}  
{{ projectAddressForm }}  
{{ projectForm }}  
#这是FormMixin父类帮我们生成的  
{{ form }}  
```

至此，我们可以只用几十行代码就完成复杂的功能，代码逻辑也清晰可控。

从这篇文章里也可以看得出，django实在是快速开发网站的必备神器！当然，快速不代表不能够支撑大并发的应用，instagram这个很火的服务就是用django写的。由于python和django过于灵活，都将要求django的开发者们唯有更资深才能写出生产环境下的服务。