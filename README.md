# django-tutorial-14th


##Part 1    


###1. 프로젝트 및 앱 생성

```
$ django-admin startproject mysite #프로젝트 생성
$ python manage.py runserver #개발 서버 실행
$ python manage.py startapp polls #설문조사 앱 생성
```

###2. view 작성

```
#polls/views.py
from django.http import HttpResponse
def index(request):
    return HttpResponse("Hello, world. You're at the polls index.")
```
* view를 호출하려면 연결된 URL이 필요하므로 URLconf를 생성하여 polls에 urls.py 생성한다

```
#polls/urls.py
from django.urls import path
from . import views
urlpatterns = [
    path('', views.index, name='index'),
]
```
* 최상위 URLconf 에서 polls.urls 모듈을 바라보게 설정
```
#mysite/urls.py
from django.contrib import admin
from django.urls import include, path
urlpatterns = [
    path('polls/', include('polls.urls')),
    path('admin/', admin.site.urls),
]
```
- *include()* 함수는 다른 URLconf들을 참조할 수 있도록 도와주며 Django가 함수 *include()* 를 만나게 되면, URL의 그 시점까지 일치하는 부분을 잘라내고, 남은 문자열 부분을 후속 처리를 위해 include 된 URLconf로 전달한다
* * *


##Part 2          
 

###1. 모델
* 모델이란 부가적인 메타데이터를 가진 데이터베이스의 구조이다
* 데이터베이스의 각 필드는 Field 클래스의 인스턴스로서 표현되며 각 필드가 어떤 자료형을 가질 수 있는지를 Django 에게 말해준다.

### 2. migration
* makemigrations은 모델의 변경사항을 migration으로 저장시키고 싶다는 것을 Django에게 알려준다
```
$ python manage.py makemigrations polls
```
* migrate 명령은 아직 적용되지 않은 migration을 모두 수집해 이를 실행하며 모델에서의 변경 사항들과 데이터베이스의 스키마의 동기화가 이루어집니다.
```
$ python manage.py migrate
```
### 3. 모델 생성

- Question model
```
class Question(models.Model):
    question_text = models.CharField(max_length=200) #CharField => 문자(character) 필드
    pub_date = models.DateTimeField('date published') # DateTimeField => 날짜와 시간(datetime) 필드 
    def __str__(self):
        return self.question_text
    def was_published_recently(self): #시간 관련 커스텀 메소드
        return self.pub_date >= timezone.now() - datetime.timedelta(days=1)
```

- Choice model
```
class Choice(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)
    def __str__(self): #객체를 표현하는 메소드
        return self.choice_text
```
    
* * *


##Part 3     


###1. View가 실제로 뭔가를 하도록 만들기
* HttpResponse - 객체를 반환
  ```
  #polls.views.py
  from django.http import HttpResponse
  ...
    return HttpResponse(output)
  ```
* render() -  HttpResponse 객체와 함께 돌려주는 구문을 쉽게 표현할 수 있도록 하는 단축 기능
  ```
  #polls.views.py
  from django.shortcuts import render
  ...
  return render(request, 'polls/index.html', context)
  ```
* Http404 - 404에러 일으키기
  ```
  from django.http import Http404
  from django.shortcuts import render

  from .models import Question
  ...
    def detail(request, question_id):
        try:
            question = Question.objects.get(pk=question_id)
        except Question.DoesNotExist:
            raise Http404("Question does not exist")
        return render(request, 'polls/detail.html', {'question': question})
  ```
* get_object_or_404() -  Http404 예외를 발생시키는 단축기능
    ```
    from django.shortcuts import get_object_or_404, render
    
    from .models import Question
    ...
    def detail(request, question_id):
        question = get_object_or_404(Question, pk=question_id)
        return render(request, 'polls/detail.html', {'question': question})
    ```
  

###2. 템플릿에서 하드코딩된 URL 제거  
* 템플릿에 링크를 적으면, 이 링크는 다음과 같이 부분적으로 하드코딩된다
    ```
    #polls/index.html 
    <li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
    ```
* polls.urls 모듈의 path() 함수에서 인수의 이름을 정의했으므로, {% url %} template 태그를 사용하여 url 설정에 정의된 특정한 URL 경로들의 의존성을 제거할 수 있다.
    ```
    <li><a href="{% url 'detail' question.id %}">{{ question.question_text }}</a></li>
    ```
  
* * *
 

##Part 4      


###1. form 요소 사용
```
<form action="{% url 'polls:vote' question.id %}" method="post">
{% csrf_token %}
{% for choice in question.choice_set.all %}
    <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}">
    <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br>
{% endfor %}
<input type="submit" value="Vote">
</form>
```
* form으로 제출된 데이터를 처리하고 그 데이터로 vote() 함수를 사용하여 정보를 저장한다.
    ```
  def vote(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    try:
        selected_choice = question.choice_set.get(pk=request.POST['choice']) # 선택된 설문의 ID를 문자열로 반환
    except (KeyError, Choice.DoesNotExist):
        return render(request, 'polls/detail.html', {
            'question': question,
            'error_message': "You didn't select a choice.",
        })
    else:
        selected_choice.votes += 1
        selected_choice.save()
        return HttpResponseRedirect(reverse('polls:results', args=(question.id,)))
  ```
  

###2. 제너릭 뷰 사용하기

* 제너릭 뷰는 일반적인 패턴을 추상화하여 앱을 작성하기 위해 Python 코드를 작성하지 않아도 된다.
* **ListView** - 개체 목록 표시 추상화
    
        ```
        class IndexView(generic.ListView):
        template_name = 'polls/index.html' # "polls/index.html" 템플릿을 사용하기 위해 ListView 에 template_name 를 전달
        context_object_name = 'latest_question_list'
    
        def get_queryset(self):
            """Return the last five published questions."""
            return Question.objects.order_by('-pub_date')[:5]
       ```
* **DetailView** - 특정 개체 유형에 대한 세부 정보 페이지 표시
    ```
    class DetailView(generic.DetailView):
    model = Question
    template_name = 'polls/detail.html' #  <app name>/<model name>_detail.html 템플릿을 사용
   ```