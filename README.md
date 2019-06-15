# 開発手順

## 事前準備
```py
mkdir project
cd project
pip install django django-crispy-forms django-filter
django-admin startproject project .
manage.py startapp app
```

## 手順1. 必要なファイルの作成

```
project
│  manage.py
│  
├─app
│  │  admin.py
│  │  apps.py 
│  │  ★filters.py 
│  │  ★forms.py 
│  │  models.py
│  │  tests.py
│  │  ★urls.py 
│  │  views.py
│  │  __init__.py
│  ├─★static 
│  │  └─★app
│  │      ├─★css 
│  │      │      ★app.css 
│  │      └─★js 
│  │          │  ★app.js 
│  │          └─ ★plugins 
│  │              └─ ★responsive-paginate.js
│  │             
│  ├─migrations
│  │      __init__.py
│  ├─★templates  
│  │  └─★app  
│  │          ★item_card.html 
│  │          ★item_confirm_delete.html 
│  │          ★item_detail.html 
│  │          ★item_filter.html 
│  │          ★item_form.html 
│  │          ★_base.html 
│  │          ★_pagination.html 
│  └─★templatetags 
│          ★item_extras.py 
└─project
        settings.py
        urls.py
        wsgi.py
        __init__.py
```

## 手順２．設定ファイル編集

```py :project/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'crispy_forms',  #追加
    'app', #追加
]
```

```python :project/settings.py
LANGUAGE_CODE = 'ja-JP'

TIME_ZONE = 'Asia/Tokyo'
```

ファイルの最後に以下を追記

```py :project/settings.py
# 管理サイトのログイン機能を通常のログイン機能として使う
LOGIN_URL='admin:login'
LOGOUT_REDIRECT_URL='/'

# django-crispy-forms 設定
CRISPY_TEMPLATE_PACK = 'bootstrap4'
```

## 手順３．モデル作成

```py :app/models.py
from django.db import models
from django.core import validators


class Item(models.Model):

    SEX_CHOICES = (
        (1, '男性'),
        (2, '女性'),
    )

    name = models.CharField(
        verbose_name='名前',
        max_length=200,
    )
    age = models.IntegerField(
        verbose_name='年齢',
        validators=[validators.MinValueValidator(1)],
        blank=True,
        null=True,
    )
    sex = models.IntegerField(
        verbose_name='性別',
        choices=SEX_CHOICES,
        default=1
    )
    memo = models.TextField(
        verbose_name='備考',
        max_length=300,
        blank=True,
        null=True,
    )
    created_at = models.DateTimeField(
        verbose_name='登録日',
        auto_now_add=True
    )

    # 以下は管理サイト上の表示設定
    def __str__(self):
        return self.name

    class Meta:
        verbose_name = 'アイテム'
        verbose_name_plural = 'アイテム'
```

## 手順４．データベース作成

```bash
manage.py makemigrations
manage.py migrate
```


## 手順５．管理サイト用の設定

```py :app/admin.py
from django.contrib import admin
from .models import Item

@admin.register(Item)
class ItemAdmin(admin.ModelAdmin):
    pass
```

```py :app/apps.py
from django.apps import AppConfig

class SampleAppConfig(AppConfig):
    name = 'app'
    verbose_name = 'アプリ'
```

## 手順６．フォーム作成

```py :app/forms.py
from django import forms
from .models import Item


class ItemForm(forms.ModelForm):

    class Meta:
        model = Item
        fields = ('name','age','sex','memo')
        widgets = {
                    'name': forms.TextInput(attrs={'placeholder':'記入例：山田　太郎'}),
                    'age': forms.NumberInput(attrs={'min':1}),
                    'sex': forms.RadioSelect(),
                    'memo': forms.Textarea(attrs={'rows':4}),
                  }
```

## 手順７．フィルター作成

検索フォームを生成する設定クラスを定義します。
このクラスは追加パッケージ「django-filter」の設定となります。ここで定義した内容は検索一覧画面の検索フォームに反映されます。

```py :app/filters.py
from django_filters import filters
from django_filters import FilterSet
from .models import Item


class MyOrderingFilter(filters.OrderingFilter):
    descending_fmt = '%s （降順）'


class ItemFilter(FilterSet):

    name = filters.CharFilter(label='氏名', lookup_expr='contains')
    memo = filters.CharFilter(label='備考', lookup_expr='contains')

    order_by = MyOrderingFilter(
        # tuple-mapping retains order
        fields=(
            ('name', 'name'),
            ('age', 'age'),
        ),
        field_labels={
            'name': '氏名',
            'age': '年齢',
        },
        label='並び順'
    )

    class Meta:

        model = Item
        fields = ('name', 'sex', 'memo',)
```

## 手順８．ビュー作成

**app/views.pyの作成**

```py :app/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import ListView, DetailView
from django.views.generic.edit import CreateView, UpdateView, DeleteView
from django_filters.views import FilterView

from .models import Item
from .filters import ItemFilter
from .forms import ItemForm


# Create your views here.
# 検索一覧画面
class ItemFilterView(LoginRequiredMixin, FilterView):
    model = Item
    filterset_class = ItemFilter
    # デフォルトの並び順を新しい順とする
    queryset = Item.objects.all().order_by('-created_at')

    # クエリ未指定の時に全件検索を行うために以下のオプションを指定（django-filter2.0以降）
    strict = False

    # 1ページあたりの表示件数
    paginate_by = 10

    # 検索条件をセッションに保存する or 呼び出す
    def get(self, request, **kwargs):
        if request.GET:
            request.session['query'] = request.GET
        else:
            request.GET = request.GET.copy()
            if 'query' in request.session.keys():
                for key in request.session['query'].keys():
                    request.GET[key] = request.session['query'][key]

        return super().get(request, **kwargs)


# 詳細画面
class ItemDetailView(LoginRequiredMixin, DetailView):
    model = Item


# 登録画面
class ItemCreateView(LoginRequiredMixin, CreateView):
    model = Item
    form_class = ItemForm
    success_url = reverse_lazy('index')


# 更新画面
class ItemUpdateView(LoginRequiredMixin, UpdateView):
    model = Item
    form_class = ItemForm
    success_url = reverse_lazy('index')


# 削除画面
class ItemDeleteView(LoginRequiredMixin, DeleteView):
    model = Item
    success_url = reverse_lazy('index')
```

**app/urls.pyの作成**

```py :app/urls.py
from django.urls import path
from .views import ItemFilterView, ItemDetailView, ItemCreateView, ItemUpdateView, ItemDeleteView


urlpatterns = [
    # 一覧画面
    path('',  ItemFilterView.as_view(), name='index'),
    # 詳細画面
    path('detail/<int:pk>/', ItemDetailView.as_view(), name='detail'),
    # 登録画面
    path('create/', ItemCreateView.as_view(), name='create'),
    # 更新画面
    path('update/<int:pk>/', ItemUpdateView.as_view(), name='update'),
    # 削除画面
    path('delete/<int:pk>/', ItemDeleteView.as_view(), name='delete'),
]
```

## 手順９．テンプレート作成

・app/templatetags/item_extras.py

```py :app/templatetags/item_extras.py
from django import template

register = template.Library()

@register.simple_tag
def url_replace(request, field, value):
    dict_ = request.GET.copy()
    dict_[field] = value
    return dict_.urlencode()
```

共通テンプレート

```html :app/templates/app/_base.html
{% load static %}
<!DOCTYPE html>
<html lang="ja">

<head>
    <!-- Required meta tags always come first -->
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <meta http-equiv="x-ua-compatible" content="ie=edge">
    <title>アプリケーション名</title>
    <!-- Bootstrap CSS -->
    <link href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-Gn5384xqQ1aoWXA+058RXPxPg6fy4IWvTNh0E263XmFcJlSAwiGgFAW/dAiS6JXm"
        crossorigin="anonymous">
    <link href="{% static "app/css/app.css" %}" rel="stylesheet">
</head>

<body>
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
        <a class="navbar-brand" href="#">アプリケーション名</a>
        <button type="button" class="navbar-toggler" data-toggle="collapse" data-target="#Navber" aria-controls="Navber" aria-expanded="false"
            aria-label="ナビゲーションの切替">
            <span class="navbar-toggler-icon"></span>
        </button>

        <div class="collapse navbar-collapse" id="Navber">
            <ul class="navbar-nav mr-auto">
                <li class="nav-item">
                    <a class="nav-link" href="{% url 'admin:index'%}">管理サイト</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="{% url 'admin:logout'%}">ログアウト</a>
                </li>
            </ul>
        </div>
        <!-- /.navbar-collapse -->
    </nav>
    {% block content %} 
    {% endblock %}

    <!-- jQuery first, then Tether, then Bootstrap JS. -->
    <script src="https://code.jquery.com/jquery-3.3.1.min.js" integrity="sha256-FgpCb/KJQlLNfOu91ta32o/NMZxltwRo8QtmkMRdAu8="
        crossorigin="anonymous"></script>
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/js/bootstrap.min.js"></script>
    <script src="{% static "app/js/plugins/responsive-paginate.js" %}"></script>
    <script src="{% static "app/js/app.js" %}"></script>
</body>

</html>
```

・app/templates/app/_pagination.html

```html :app/templates/app/_pagination.html
{% load item_extras %}
<ul class="pagination">
    {% if page_obj.has_previous %}
        <li class="page-item pagination-prev">
            <a class="page-link" href="?{% url_replace request 'page' page_obj.previous_page_number %}">&laquo;</a>
        </li>
    {% else %}
        <li class="disabled page-item pagination-next">
            <span class="page-link">&laquo;</span>
        </li>
    {% endif %}
    {% for page in page_obj.paginator.page_range %}
        {% if page %}
            {% ifequal page page_obj.number %}
                <li class="active page-item">
                    <span class="page-link">{{ page }}
                        <span class="page-link sr-only">(current)</span>
                    </span>
                </li>
            {% else %}
                <li class="page-item">
                    <a class="page-link" href="?{% url_replace request 'page' page %}">{{ page }}</a>
                </li>
            {% endifequal %}
        {% endif %}
    {% endfor %}
    {% if page_obj.has_next %}
        <li class="page-item pagination-next">
            <a class="page-link" href="?{% url_replace request 'page' page_obj.next_page_number %}">&raquo;</a>
        </li>
    {% else %}
        <li class="disabled page-item pagination-next">
            <span class="page-link ">&raquo;</span>
        </li>
    {% endif %}
</ul>
```

・app/templates/app/item_card.html

データの詳細を表示。詳細画面と削除画面で共通利用

```html :app/templates/app/item_card.html
<div class="row">
    <div class="col-3">
        <p>名前</p>
    </div>
    <div class="col-9">
        <p>{{ item.name }}</p>
    </div>
</div>
<div class="row">
    <div class="col-3">
        <p>年齢</p>
    </div>
    <div class="col-9">
        <p>{{ item.age }}</p>
    </div>
</div>
<div class="row">
    <div class="col-3">
        <p>性別</p>
    </div>
    <div class="col-9">
        <p>{{ item.get_sex_display }}</p>
    </div>
</div>
<div class="row">
    <div class="col-3">
        <p>備考</p>
    </div>
    <div class="col-9">
        <p>{{ item.memo|linebreaksbr }}</p>
    </div>
</div>
<div class="row">
    <div class="col-3">
        <p>登録日</p>
    </div>
    <div class="col-9">
        <p>{{ item.created_at|date:"Y/m/d G:i:s" }}</p>
    </div>
</div>
```

・app/templates/app/item_confirm_delete.html

削除画面

```html :app/templates/app/item_confirm_delete.html
{% extends "./_base.html" %}
<!--  -->
{% block content %}
<div class="container">
    <h2 class="text-center">データ削除</h2>
    <p>このデータを削除します。よろしいですか？</p>

    <form action="" method="post">
        {% csrf_token %}
        <div class="row">
            <div class="col-12">
                <div class="float-right">
                    <a class="btn btn-outline-secondary" href="{% url 'index' %}">戻る</a>
                    <input type="submit" class="btn btn-outline-secondary" value="削除" />
                </div>
            </div>
        </div>
        {% include "./item_card.html" %}
        <div class="row">
            <div class="col-12">
                <div class="float-right">
                    <a class="btn btn-outline-secondary" href="{% url 'index' %}">戻る</a>
                    <input type="submit" class="btn btn-outline-secondary" value="削除" />
                </div>
            </div>
        </div>
    </form>
</div>
{% endblock %}
```

・app/templates/app/item_detail.html

詳細画面

```html :app/templates/app/item_detail.html
{% extends "./_base.html" %}
{% block content %}
<div class="container">
    <h2 class="text-center">詳細表示</h2>
    <div class="row">
        <div class="col-12">
            <a class="btn btn-outline-secondary float-right" href="{% url 'index' %}">戻る</a>
        </div>
    </div>
    <!--  -->
    {% include "./item_card.html" %}
    <div class="row">
        <div class="col-12">
            <a class="btn btn-outline-secondary float-right" href="{% url 'index' %}">戻る</a>
        </div>
    </div>
</div>
{% endblock %}
```

・app/templates/app/item_filter.html

検索一覧画面

```html :app/templates/app/item_filter.html
{% extends "./_base.html" %}
{% block content %} 
{% load crispy_forms_tags %}
<div class="container">
    <div id="myModal" class="modal fade" tabindex="-1" role="dialog">
        <div class="modal-dialog" role="document">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title">検索条件</h5>
                    <button type="button" class="close" data-dismiss="modal" aria-label="閉じる">
                        <span aria-hidden="true">&times;</span>
                    </button>
                </div>
                <form id="filter" method="get">
                    <div class="modal-body">
                        {{filter.form|crispy}}
                    </div>
                </form>
                <div class="modal-footer">
                    <a class="btn btn-outline-secondary" data-dismiss="modal">戻る</a>
                    <button type="submit" class="btn btn-outline-secondary" form="filter">検索</button>
                </div>
            </div>
        </div>
    </div>
    <div class="row">
        <div class="col-12">
            <a class="btn btn-secondary filtered" style="visibility:hidden" href="/?page=1">検索を解除</a>
            <div class="float-right">
                <a class="btn btn-outline-secondary" href="{% url 'create' %}">新規</a>
                <a class="btn btn-outline-secondary" data-toggle="modal" data-target="#myModal" href="#">検索</a>
            </div>
        </div>
    </div>

    <div class="row" >
        <div class="col-12">
            {% include "./_pagination.html" %}
        </div>
    </div>

    <div class="row">
        <div class="col-12">
            <ul class="list-group">
                {% for item in item_list %}
                <li class="list-group-item">
                    <div class="row">
                        <div class="col-3">
                            <p>名前</p>
                        </div>
                        <div class="col-9">
                            <p>{{ item.name }}</p>
                        </div>
                    </div>
                    <div class="row">
                        <div class="col-3">
                            <p>登録日</p>
                        </div>
                        <div class="col-9">
                            <p>{{item.created_at|date:"Y/m/d G:i:s"}}</p>
                        </div>
                    </div>
                    <div class="row">
                        <div class="col-12">
                            <div class="float-right">
                                <a class="btn btn-outline-secondary " href="{% url 'detail' item.pk %}">詳細</a>
                                <a class="btn btn-outline-secondary " href="{% url 'update' item.pk %}">編集</a>
                                <a class="btn btn-outline-secondary " href="{% url 'delete' item.pk %}">削除</a>
                            </div>
                        </div>
                    </div>
                </li>
                {% empty %}
                <li class="list-group-item">
                    対象のデータがありません
                </li>
                {% endfor %}
            </ul>
        </div>
    </div>
    <div class="row" >
        <div class="col-12">
            <div class="float-right">
                <a class="btn btn-outline-secondary" href="{% url 'create' %}">新規</a>
                <a class="btn btn-outline-secondary" data-toggle="modal" data-target="#myModal" href="#">検索</a>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

・app/templates/app/item_form.html

登録画面・更新画面（共通）

```html :app/templates/app/item_form.html
{% extends "./_base.html" %}
{% load crispy_forms_tags %}
{% block content %}
{{ form.certifications.errors }}
<div class="container">
    <div class="row">
        <div class="col-12">
            <h2 class="text-center">データ入力</h2>
        </div>
    </div>
    <div class="row">
        <div class="col-12">
            <div class="float-right">
                <a class="btn btn-outline-secondary" href="{% url 'index' %}">戻る</a>
                <a class="btn btn-outline-secondary save" href="#">保存</a>
            </div>
        </div>
    </div>
    <div class="row">
        <div class="col-12">
            <form method="post" id="myform">
                {%crispy form%}
            </form>
        </div>
    </div>
    <div class="row">
        <div class="col-12">
            <div class="float-right">
                <a class="btn btn-outline-secondary" href="{% url 'index' %}">戻る</a>
                <a class="btn btn-outline-secondary save" href="#">保存</a>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

共通css/js

・app/static/app/css/app.css

```css :app/static/app/css/app.css
.row{
    margin-top:3px;
    margin-bottom:3px;
}
```

rPageについて
ページネーション（特定ページへのリンク）が多いときに、画面幅からあふれる分を自動的に省略するプラグインです。

```js :app/static/app/js/app.js
// 入力フォームでリターンキー押下時に送信させない
$('#myform').on('sumbit', function (e) {
    e.preventDefault();
})

// 連続送信防止
$('.save').on('click', function (e) {
    $('.save').addClass('disabled');
    $('#myform').submit();
})

// [検索を解除] の表示制御
conditions = $('#filter').serializeArray();
$.each(conditions, function(){
    if(this.value){
        $('.filtered').css('visibility','visible')
    }
})

// ページネーションのレスポンシブ対応
// https://auxiliary.github.io/rpage/
$(".pagination").rPage();
```

## ３．カスタマイズ作業

### ３－１．修正ファイルは３つだけ

```
project_root
├ config              # 設定ディレクトリ
│ │ settings.py       # 設定
│ │ urls.py           # ルーティング設定
│ └ wsgi.py           # 本番環境での起動ファイル
│
├ app                 # アプリ ディレクトリ
│ ├ migrations        # データベース定義
│ ├ static            # 静的ファイル(img/js/css)
│ ├ templates         
│ │ └ app             # 画面テンプレート
│ │   │ _base.html                 # 共通部品
│ │   │ _pagination.html           # ページネーション部品
│ │   │ item_confirm_delete.html   # 削除画面
│ │   │ item_detail.html           # 詳細画面
│ │   │ item_detail_contents.html  # 詳細・削除画面 共通部品 【★】
│ │   │ item_filter.html           # 検索一覧画面 【★】
│ │   └ item_form.html             # 新規登録・更新画面
│ │  
│ ├ templatetags      # テンプレートタグ
│ │ admin.py          # 管理アプリ用定義
│ │ apps.py           # アプリ設定
│ │ filters.py        # 検索
│ │ forms.py          # 入力フォーム
│ │ models.py         # モデル【★】
│ │ tests.py          # テスト
│ │ urls.py           # ルーティング設定
│ └ views.py          # ビュー(MVCでのコントローラ)
│
├ users               # ユーザー管理用アプリ ディレクトリ
│
└ manage.py           # 管理コマンド
```

### ３－２．モデルファイルを修正する

```py :app/models.py
from django.db import models

from users.models import User


class Item(models.Model):
    """
    データ定義クラス
      各フィールドを定義する
    参考：
    ・公式 モデルフィールドリファレンス
    https://docs.djangoproject.com/ja/2.1/ref/models/fields/
    """

    # サンプル項目1 文字列
    sample_1 = models.CharField(
        verbose_name='サンプル項目1_文字列',
        max_length=20,
        blank=True,
        null=True,
    )

    # サンプル項目2 数値
    sample_2 = models.IntegerField(
        verbose_name='サンプル項目2_数値',
        blank=True,
        null=True,
    )

    # サンプル項目3 ブール値
    sample_3 = models.BooleanField(
        verbose_name='サンプル項目3_ブール値',
    )

    # サンプル項目4 選択肢
    sample_choice = (
        (1, '選択１'),
        (2, '選択２'),
        (3, '選択３'),
    )

    sample_4 = models.IntegerField(
        verbose_name='サンプル項目4_選択肢',
        choices=sample_choice,
        blank=True,
        null=True,
    )

    # サンプル項目5 日付
    sample_5 = models.DateField(
        verbose_name='サンプル項目5 日付',
        blank=True,
        null=True,
    )

    # 以下、管理項目

    # 作成者(ユーザー)
    created_by = models.ForeignKey(
        User,
        verbose_name='作成者',
        blank=True,
        null=True,
        related_name='CreatedBy',
        on_delete=models.CASCADE,
        editable=False,
    )

    # 作成時間
    created_at = models.DateTimeField(
        verbose_name='作成時間',
        blank=True,
        null=True,
        editable=False,
    )

    # 更新者(ユーザー)
    updated_by = models.ForeignKey(
        User,
        verbose_name='更新者',
        blank=True,
        null=True,
        related_name='UpdatedBy',
        on_delete=models.CASCADE,
        editable=False,
    )

    # 更新時間
    updated_at = models.DateTimeField(
        verbose_name='更新時間',
        blank=True,
        null=True,
        editable=False,
    )

    def __str__(self):
        """
        リストボックスや管理画面での表示
        """
        return self.sample_1

    class Meta:
        """
        管理画面でのタイトル表示
        """
        verbose_name = 'サンプル'
        verbose_name_plural = 'サンプル'
```

### ３－３．データベースに反映する

```bash
python manage.py makemigrations
python manage.py migrate
```

### ３－４．検索一覧・詳細画面の修正

```html :item_detail_contents.html
<div class="row">
    <div class="col-5 col-sm-3">
        <p>1_文字列</p>
    </div>
    <div class="col-7 col-sm-9">
        <p>{{  item.sample_1|default_if_none:"未入力"  }}</p>
    </div>
</div>
```