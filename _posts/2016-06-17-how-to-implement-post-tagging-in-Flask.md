---
layout: default
title: 在Flask中实现博客文章标签功能
---

# 在Flask中实现博客文章标签功能

### Flask简介

Flask是一个基于Python的微型后端框架。它不像Django等其他基于Python的大型后端框架那样集成了大量的组件，十分臃肿，而是只包含一个基本的核心，其他的功能由用户按自己的需求来构建，非常自由，也十分适合个人编写如博客等的网站。

这篇文章适合有Flask编程基础的朋友阅读，如果是没有Flask基础的朋友，建议你看一看网上关于Flask的教程，有编程基础的基本一天内都可以上手。

下面这本书是我认为比较好的入门教程：

[Flask Web开发：基于Python的Web应用开发实战](https://book.douban.com/subject/26274202/)

这本书是外国人写的一本书，已经出版，也由人民邮电出版社翻译出版，书的内容是围绕一个小型微博应用来写的，用词等都很通俗易懂，用到的技术也十分适合简单，适合入门者学习，个人十分推荐。

### 博客文章分类和标签

文章分类是博客中十分常见的功能，把相关的文章放在一起，读者可以快捷的找到自己想看的内容，增加阅读体验，也方便博客管理者对于网站的管理。要实现这样的功能，最简单的办法就是在数据库中新建一个表，用来储存每个类别，如C，Python，HTML等，然后把每篇文章归类到这些类别下就可以了，表中每一项对应若干篇文章，这也是数据库中常用的一对多关系。

但有时有一些文章是涵盖多个类别的，例如一篇关于C与Python交互的文章，我们很难将它单独归类于C或Python类别下，为它单独创建一个“C与Python”的类别又有点浪费。标签功能就是解决这个问题的好方法，一篇文章可以对应多个标签，如之前的例子，可以同时有C和Python的标签，那样就可以在文章中找到相关的其他文章，方便读者阅读。甚至可以用标签功能直接取代分类功能，显得更方便。

![常见博客网站分类与标签](https://github.com/phileong/phileong.github.io/raw/master/images/category-and-tag-in-common-blog.jpg)

### 数据库设计

在实际编写代码之前，我们先说一下分类和标签在数据库中的不同点在那里。

分类是典型的一对多关系，即一个表中的每项对应另一个表中的若干项，如C语言的分类下有多篇文章，我们可以经由A表中的C语言项查找B表中对应的文章，这就是一对多，但这有一个缺陷，就是我们没办法给文章添加多个分类。而标签是多对多的关系，即A表中的每一项对应B表中的若干项，同时B表中的每一项也对应着A表中的若干项，这样就可以解决这个问题，这就是数据库中所谓多对多的关系。

![一对一关系](https://github.com/phileong/phileong.github.io/raw/master/images/one-to-one-relationship.jpg)

![一对多关系](https://github.com/phileong/phileong.github.io/raw/master/images/one-to-many-relationship.jpg)

我们可以看到，两者在实现上是十分相似的，只是一对多关系要比一对多关系要多使用一个表作为连接。所以在代码实现上有点复杂，但对有开发经验的人来说也是十分简单的。

在具体编写代码之前，我建议读者看一看上面提到的教程，因为我接下来不会逐一介绍Flask的安装和基本用法等内容，而是直接从数据库交互写起，所以需要读者有一定的Flask基础。你也可以只看数据库一章，了解Flask-SQLAlchemy插件的使用，我们主要使用此插件，其余插件不会也基本不成问题。废话不多说，接下来就是具体代码实现。

### 代码实现

首先，我们先写一个基本的Flask应用，建立一个test.py文件，填入以下代码，下面这段代码只是一个简单的HelloWorld页面，十分简单，之后我们在这个基础上编写实现代码。

后面的代码除了一些关键代码，其余重复的都会省略，省略的代码用**...**来取代，这样也方便我们关注重要的内容，但我也会保留一些代码用以定位，让读者知道在哪里添加新的代码，读者不用担心。


    from flask import Flask
    
    app = Flask(__name__)
    
    @app.route('/')
    def index():
        return "<h1>Hello, world!</h1>"
    
    if __name__ == "__main__":
        app.run(debug=True)
        

----------------------------------------

接下来是编写数据库类，我们使用Flask-SQLAlchemy，数据库使用SQLite软件，这个数据库软件是Python默认包含的，所以不用额外安装。


    import os  # 导入os模块
    from flask import Flask
    from flask_sqlalchemy import SQLAlchemy  # 导入数据库交互插件
    
    basedir = os.path.abspath(os.path.dirname(__file__))  # 获取应用根目录
    
    app = Flask(__name__)
    
    app.config["SQLALCHEMY_DATABASE_URI"] = \
        "sqlite:///" + os.path.join(basedir, "data.sqlite")  # 设置数据库文件储存位置
    db = SQLAlchemy(app)  # 建立数据库实例，我们通过这个实例与数据库交互
    
    # ...
    

注意：可能你运行时会报关于SQLALCHMEY_TRACK_MODIFICATIONS的警告，只要添加一行代码就可以解决，当然，你不添加也没有问题，不影响代码功能。

    app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = True
    
----------------------------------------

建立了数据库实例之后，我们就可以编写类了，这些类每个都与数据库中的一个表对应，我们用这些类对象操作数据库中的表。我们会用到Flask-SQLAlchemy提供的一个基类Model，这个基类包含了常用的数据库操作，如插入，修改和删除行等。

![数据库设计](https://github.com/phileong/phileong.github.io/raw/master/images/one-to-many-relationship.jpg)

    
    # ...
    
    db = SQLAlchemy(app)
    
    class Post(db.Model):
        __tablename__ = "posts"  # 数据库表名
        id = db.Column(db.Integer, primary_key=True)  # 主键
        title = db.Column(db.String(120))
        
    class Tag(db.Model):
        __tablename__ = "tags"
        id = db.Column(db.Integer, primary_key=True)
        name = db.Column(db.String(120))

    class PostTagRelationship(db.Model):
        __tablename__ = "post_tag_relationships"
        id = db.Column(db.Integer, primary_key=True)
        post_id = db.Column(db.Integer, db.ForeignKey("posts.id"))  # 外键
        tag_id = db.Column(db.Integer, db.ForeignKey("tags.id"))
        
    # ...
        
        
----------------------------------------

这样，我们的数据库模型就差不多完成了，但如果只使用以上的代码在使用上会有些不方便，我们在添加一些东西。


    # ...
    
    class Post(db.Model):
        # ...
        tag_relationships = db.relationship("PostTagRelationship", backref="post")
    
    class Tag(db.Model):
        # ...
        post_relationships = db.relationship("PostTagRelationship", backref="tag")
        
    # ...
    
    
添加到Post和Tag类中的tag_relationships和post_relationships属性代表这个关系的面向对象视角。

就是说，对于一个Post类，访问它的tag_relationships属性会返回一个列表，这个列表是由post_tag_relationships表中所有post_id与其本身相同的项组成，每个项都是一个PostTagRelationship实例。

db.relationship()中的backref参数向PostTagRelationsip模型中添加一个post属性，这个属性可以代替post_id访问对应的Post模型，此时post获取的是Post模型对象，而不是post_id的值。

----------------------------------------

好了，代码已经完成，我们现在就可以双向查询了，但在测试之前我们还要给每个类添加__repr__()方法，返回一个可读的字符串，方便调试和测试。


    # ...
    
    class Post(db.Model):
        # ...
        def __repr__(self):
            return "<Post %r>" % self.title
    
    class Tag(db.Model):
        # ...
        def __repr__(self):
            return "<Tag %r>" % self.name
            
    class PostTagRelationship(db.Model):
        # ...
        def __repr__(self):
            return "<Relationship %r>" % self.id
            
    # ...
    
    
### 功能测试

为了方便调试，我们使用Flask-Script插件，安装方法如下：

    (venv) $ pip install flask-script
    
在添加以下代码：


    # ...
    
    from flask_sqlalchemy import SQLAlchemy
    from flask_script import Manager
    
    # ...
    
    db = SQLAlchemy(app)
    manager = Manager(app)
    
    # ...
    
    if __name__ == "__main__":
        manager.run()
        
        
把最后一行代码

    app.run(debug=True)
        
修改为：

    manager.run()
    
然后我们就可以使用以下命令来进入命令行对程序进行调试。

    (venv) $ python test.py shell

----------------------------------------

以下命令是在Python命令行中进行的。

首先我们要创建一个数据库，导入所有的类，然后把两个项C与Python插入tags表中

    >>> from test import db
    >>> db.create_all()
    >>>
    >>> from test import Post, Tag, PostTagRelationship
    >>> t1 = Tag(name='C')
    >>> t2 = Tag(name="Python")
    >>>
    >>> db.session.add(t1)
    >>> db.session.add(t2)
    >>> db.session.commit()
    
运行以上命令，然后再次查看t1和t2的id属性，现在它们应该已经赋值了，证明这两项已经插入表中。接下来就是展示如何为文章添加标签了。

我们先添加一篇文章，提交到数据库。然后在post_tag_relationships表中添加一个项，我们为文章添加一个C标签。

    >>> p1 = Post(title="Post One")
    >>> db.session.add(p1)
    >>> db.session.commit()
    >>>
    >>> r = PostTagRelationship(post=p1, tag=t1)
    >>> db.session.add(r)
    >>> db.session.commit()
    
现在，文章一已经在标签C下了，我们可以查看结果。

    >>> t = Tag.query.filter_by(name="C").first()
    >>> rs = PostTagRelationship.query.filter_by(tag=t).all()
    >>> rs[0].post
    <Post 'Post One'>
    
先给出一个标签，查找post_tag_relationships表中所有与该标签有关的项，然后查看这些项对应的文章。

我们在尝试一下查找一篇文章的所有标签。

    >>> r2 = PostTagRelationship(post=p1, tag=t2)
    >>> db.session.add(r2)
    >>> db.session.commit()
    >>>
    >>> p = Post.query.filter_by(id=1).first()
    >>> rs = PostTagRelationship.query.filter_by(post=p).all()
    >>> ts
    [<Relationship 1>, <Relationship 2>]
    >>> rs[0].tag
    <Tag 'C'>
    >>> rs[1].tag
    <Tag 'Python'>
    
测试成功。