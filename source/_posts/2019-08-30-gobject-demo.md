---
title:      "GObject 编程入门"
date:       2019-08-30 20:50:27
author:     "lixiang"
categories: Linux 编程
tags:
    - Glib GObject
---

> Gobject，也称Glib对象系统，是一个程序库，它可以帮助我们使用C语言编写面向对象程序，其提供了一个通用的动态类型系统（GType）、一个基本类型的实现集（如整型、枚举等）、一个基本对象类型Gobject、一个信号系统以及一个可扩展的参数/变量体系。在 GObject世界里，类是两个结构体的组合，一个是实例结构体，另一个是类结构体。GOBJECT的继承需要实现实例结构体的继承和类结构体的继承，Gobject对象的初始化可分为两个部分：类结构体初始化和实例结构体初始化。类结构体初始化函数只被调用一次，而实例结构体的初始化函数的调用次数等于对象实例化的次数。这意味着，所有对象共享的数据，可保存在类结构体中，而所有对象私有的数据，则保存在实例结构体中。

---

## 示例源码
- [gobjectdemo](https://github.com/eightplus/examples/tree/master/code/C/gobjectdemo)

## GObject编程要点简析

- 宏

  如果你浏览过使用 gobject 编写的软件源代码，你对下面出现的宏一定不会陌生。
  ```
  #define MY_TYPE_DEMO (my_demo_get_type())
  #define MY_DEMO(object) G_TYPE_CHECK_INSTANCE_CAST((object), MY_TYPE_DEMO, MyDemo)
  #define MY_DEMO_CLASS(klass) (G_TYPE_CHECK_CLASS_CAST((klass), MY_TYPE_DEMO, MyDemoClass))
  #define MY_IS_DEMO(object) G_TYPE_CHECK_INSTANCE_TYPE((object), MY_TYPE_DEMO))
  #define MY_IS_DEMO_CLASS(klass) (G_TYPE_CHECK_CLASS_TYPE((klass), MY_TYPE_DEMO))
  #define MY_DEMO_GET_CLASS(object) (G_TYPE_INSTANCE_GET_CLASS((object), MY_TYPE_DEMO, MyDemoClass))

  GType my_demo_get_type(void) G_GNUC_CONST;
  G_DEFINE_TYPE (MyDemo, my_demo, G_TYPE_OBJECT);
  ```

  这里对上面的宏不作过多阐述，简要介绍下 MY_DEMO 和 G_DEFINE_TYPE。

  MY_DEMO(object)宏的作用是将一个基类指针类型转换为 MyDemo 类的指针类型，它需要 GObject 子类的设计者提供。

  my_demo_get_type将会使用宏G_DEFINE_TYPE去实现，其中的my通常表示命名空间，demo表示对象名字，get_type为固定字段(这里提到的my和demo字段都是我的示例代码使用的，你可以自定义)。G_DEFINE_TYPE 可以让 GObject 库的数据类型系统能够识别我们所定义的 MyDemo 类类型，它接受三个参数，第一个参数是类名，即 MyDemo，第二个参数则是类的成员函数名称的前缀，例如 my_demo_get_type 函数即为 MyDemo 类的一个成员函数，"my_demo"是它的前缀，第三个参数则指明 MyDemo 类类型的父类型为G_TYPE_OBJECT。G_DEFINE_TYPE中会调用一个 g_type_register_static_simple 的函数，这个函数的作用就是将用户自己定义的类型注册到系统中，G_DEFINE_TYPE还定义了2个函数（my_demo_init 和 my_demo_class_init），需要自己实现这两个函数，这两个函数是对象的初始化函数，相当于c++的构造函数，其中第一个函数在每个对象创建的时候都会被调用，第二个函数只有在第一次创建对象的时候被调用(比如在调用g_type_class_ref的时候，如果class没有初始化，就会调用my_demo_class_init)。当然这里也可以直接使用，而是使用的实现代码，见示例代码中被注释的那一大段代码。在gobject的手册中也有相关代码的示例，如下所示（[G-DEFINE-TYPE](https://developer.gnome.org/gobject/stable/gobject-Type-Information.html#G-DEFINE-TYPE:CAPS)）：

  ```
  static void     gtk_gadget_init       (GtkGadget      *self);
  static void     gtk_gadget_class_init (GtkGadgetClass *klass);
  static gpointer gtk_gadget_parent_class = NULL;
  static void     gtk_gadget_class_intern_init (gpointer klass)
  {
    gtk_gadget_parent_class = g_type_class_peek_parent (klass);
    gtk_gadget_class_init ((GtkGadgetClass*) klass);
  }

  GType
  gtk_gadget_get_type (void)
  {
    static volatile gsize g_define_type_id__volatile = 0;
    if (g_once_init_enter (&g_define_type_id__volatile))
      {
        GType g_define_type_id =
          g_type_register_static_simple (GTK_TYPE_WIDGET,
                                         g_intern_static_string ("GtkGadget"),
                                         sizeof (GtkGadgetClass),
                                         (GClassInitFunc) gtk_gadget_class_intern_init,
                                         sizeof (GtkGadget),
                                         (GInstanceInitFunc) gtk_gadget_init,
                                         0);
        {
          const GInterfaceInfo g_implement_interface_info = {
            (GInterfaceInitFunc) gtk_gadget_gizmo_init
          };
          g_type_add_interface_static (g_define_type_id, TYPE_GIZMO, &g_implement_interface_info);
        }
        g_once_init_leave (&g_define_type_id__volatile, g_define_type_id);
      }
    return g_define_type_id__volatile;
  }
  ```

- 对象实例化
  ```
  gpointer
  g_object_new (GType object_type,
                const gchar *first_property_name,
                ...);
  ```
  g_object_new进行对象的实例化，这个函数是个可变参数的函数, 第一个参数为一个宏，是需要创建的对象的类型，当使用 g_object_new 来创建对象的时候, 这个参数是必须的，而且这个函数所创建的对象必须是GObject的子对象，在我们定义自己的对象时, 必须要在glib系统中注册自己的类型。后面的参数都是 "属性名-属性值" 的配对，即从第二个参数开始, 表示的是 object 的属性, 它们都是成对出现的，如果没有属性需要在创建对象的时候设置, 则第二个参数设置成 NULL（`g_object_new(MY_TYPE_DEMO, NULL)`），如果有属性要设置, 那么最后一个参数也要设置成 NULL（`g_object_new(MY_TYPE_DEMO, "name", "kobe", "age", 24, NULL)`）。

- 属性
  如果想 让g_object_new 函数中通过 "属性名-属性值" 结构为 Gobject 子类对象的属性进行初始化，需要完成以下工作：
    - 实现xx_xx_set_property与xx_xx_get_property函数，如示例代码中的my_demo_set_property和my_demo_get_property，完成 g_object_new 函数 "属性名-属性值" 结构向Gobject子类属性的映射；
    - 在Gobject子类的类结构体初始化函数中，让Gobject基类的两个函数指针set_property与get_property分别指向xx_xx_set_property与xx_xx_get_property,如示例代码中的object_class->set_property = my_demo_set_property和object_class->get_property = my_demo_get_property;
    - 在Gobject子类的类结构体初始化函数中，为Gobject子类安装具体对象的私有属性，g_object_class_install_property 函数用于将 GParamSpec 类型变量所包含的数据插入到 GObject 子类中，该函数的第一个参数为 GObject 子类的类结构体，第二个参数是 GParamSpec 对应的属性 ID，GObject 子类的属性 ID 是 GObject 子类设计者定义的宏或枚举类型，不能为0，所以在使用枚举类型来定义 ID 时，为了避免 0 的使用，一般设计如下所示的一个枚举类型，其中PROP_0为0,不使用。
    ```
    enum
    {
        PROP_0,
        PROP_NAME,
        PROP_AGE,
        N_PROPERTIES
    };
    ```
    g_object_class_install_property的第三个参数是要安装值向 GObject 子类中的 GParamSpec 类型的变量指针，示例代码中的如下：
    ```
    g_object_class_install_property(object_class,
                                PROP_NAME,
                                g_param_spec_string("name",
                                                  "name",
                                                  "Specify the name of demo",
                                                  "Unknown",
                                                  G_PARAM_WRITABLE | G_PARAM_CONSTRUCT_ONLY));

    g_object_class_install_property(object_class,
                                    PROP_AGE,
                                    g_param_spec_int64 ("age",
                                                     "age",
                                                     "Specify age number",
                                                     G_MININT64, G_MAXINT64, 0,
                                                     G_PARAM_WRITABLE | G_PARAM_CONSTRUCT_ONLY));
    ```

    其中获取和设置属性的具体实现代码如下：
    ```
    static void my_demo_set_property(GObject *object, guint prop_id, const GValue *value, GParamSpec *pspec)
    {
        MyDemo *self;

        g_return_if_fail(object != NULL);

        self = MY_DEMO (object);
        //self->priv = MY_DEMO_GET_PRIVATE(self);

        switch (prop_id)
        {
        case PROP_NAME:
            self->priv->name = g_value_dup_string(value);
            break;
        case PROP_AGE:
            self->priv->age = g_value_get_int64(value);
            break;
        default:
            G_OBJECT_WARN_INVALID_PROPERTY_ID(object, prop_id, pspec);
            break;
        }
    }

    static void my_demo_get_property(GObject *object, guint prop_id, GValue *value, GParamSpec *pspec)
    {
        MyDemo *self;

        g_return_if_fail(object != NULL);

        self = MY_DEMO(object);

        switch (prop_id)
        {
        case PROP_NAME:
            g_value_set_string(value, self->priv->name);
            break;
        case PROP_AGE:
            g_value_set_int64(value, self->priv->age);
            break;
        default:
            G_OBJECT_WARN_INVALID_PROPERTY_ID(object, prop_id, pspec);
            break;
        }
    }
    ```
    当然，我们也可以通过g_object_get_property 和 g_object_set_property 获取和设置属性，如下所示：
    ```
      GValue val = {0};       
      g_value_init(val, G_TYPE_POINTER);
      g_object_get_property(G_OBJECT(demo), "name", val);
      g_object_set_property(G_OBJECT(demo), "name", val);
      g_value_unset(val);
    ```

- 信号

  使用 GObject 信号机制，一般有三个步骤：
    - 信号注册 [g_signal_new](https://developer.gnome.org/gobject/stable/gobject-Signals.html#g-signal-new)，主要解决信号与数据类型的关联问题
    ```
    int signal_id = g_signal_new("broadcast_msg",
            my_demo_get_type(),/*G_OBJECT_CLASS_TYPE(kclass)*/
            G_SIGNAL_RUN_LAST,
            0,
            NULL,
            NULL,
            g_cclosure_marshal_VOID__STRING,
            G_TYPE_NONE,/*返回值，因为信号没有返回，所以为NONE*/
            1,/*参数数目*/
            G_TYPE_STRING/*参数类型*/
        );
    ```
    - 信号连接
    ```
    void handler_receive_msg(GObject *sender, char *name, gpointer data)
    {
        MyDemo *demo = G_TYPE_CHECK_INSTANCE_CAST(sender, my_demo_get_type(), MyDemo);
        printf("handler: [%s] from [%s: %ld]\n", name, demo->priv->name, demo->priv->age);
    }

    g_signal_connect(demos[i], "broadcast_msg", G_CALLBACK(handler_receive_msg), NULL);
    ```
    - 信号发射
    ```
    g_signal_emit_by_name(demos[i], "broadcast_msg", "lixiang");
    ```
