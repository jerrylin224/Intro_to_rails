指令rails new => 建立專案
問：實際上這個指令產生了哪些檔案？

  rails new suggestotron #建立一個叫“suggestotron”的專案
![](http://www.createyourownlives.com/wp-content/uploads/屏幕截图-2017-02-21-12.10.40.png)

指令rails server => 執行程式，但是也可以縮寫成rails s

  rails s

註：建立一個簡單的資料庫（表格）=>你可以仔細闡述資料庫跟表格的關係嗎？

  rails generate scaffold topic title:string description

解釋：
1.generate scaffold，告訴rails建立一個操作topics功能下所需的所有檔案
2.topic，告訴rails這個新model的名字
3.title:string，告訴rails這個topics的資料庫Table裡會有個欄位叫做title，資料格式是”string"
問：這裡生成的table名稱叫做？
4.description:text，告訴rails這個topics的資料庫Table裡會有個欄位叫做description，資料格式是“text”

  rake db:migrate #目的主要就是更新架構
（沒有更新的話網站會fail）
告訴rails要將database的架構更新，把剛剛新建的model設定放進去
一個關於rails指令的[網址](http://stackoverflow.com/questions/10301794/difference-between-rake-dbmigrate-dbreset-and-dbschemaload)


rails跟Database溝通的功能幾乎都內建在ActiveRecord::Base裡
  class Topic < ActiveRecord::Base

問：[你知道scaffold幫你做了些什麼嗎？](http://zh-tw.railsbridge.org/intro-to-rails/%E4%BD%BF%E7%94%A8%E9%B7%B9%E6%9E%B6%E4%BE%86%E5%AE%8C%E6%88%90_CRUD_%E5%8A%9F%E8%83%BD)（紀錄一下好了）


config中的routes.rb檔就是來處理路徑問題的

要把root路徑設定使用index.html.erb的方法是：
問：但是這邊比較confuse的是，routes跟draw分別是？
  Rails.application.routes.draw do
      root ‘topics#index’
  end

root ‘topics#index’是跟rails說我的首頁要指向topics#index，而topics#index是topics_controller.rb裡的index action。
config/routes.rb代表了這個網站裡的地址目錄，列出所有可以前往的頁面

scaffold會建立一個完整的CRUD，但我們不需要在votes這個model上建立CRUD，只需要它作為topics的一部分就好。（？？？

has_many & belongs_to：這兩個是用以設定不同model（table）之間的關係
1. has_many 所以一個topic has many votes，就要設定成
  class Topic < ApplicationRecord
      has_many :votes, dependent: :destroy 
  end
而後面的dependent: :destroy代表topic instance被刪除時，它所擁有的votes就會被一併刪掉，如果沒有這樣設定的話這些資料就有可能被默默地留在database裡...（這類似SQL中的on delete cascade）

2. belongs_to 而每一個vote都belongs to a topic，就要設定成
  class Vote < ApplicationRecord
      belongs_to :topic
  end


現在我們要新增一個讓使用者可以投票的功能
步驟一：在topic新增一個up_vote action
  def upvote
      @topic = Topic.find(params[:id])
      @topic.votes.create
      redirect_to topics_path
  end
@topic = Topic.find(params[:id]，透過包含在request中的params hash，找出對應的topic id，並將對應該id的topic資料儲存在@topic中
@topic.votes.create 則是會新增一筆vote並存入資料庫

步驟二：幫upvote這個action設定一個新路徑
更改resources :topics成
  resources :topics do
      members do
          post ‘upvote’
      end
  end
註：這基本上就是透過block，幫助upvote設定了一個新的路徑
註：resources :topics這個code，他實際代表的意義是什麼？
![](http://www.createyourownlives.com/wp-content/uploads/屏幕截图-2017-02-21-16.02.51.png)
步驟三：在view上新增一個按鈕
在app/views/topics/index.html.erb中加入兩行程式碼
  <td><%= pluralize(topic.votes.count, “vote" %></td>
  <td><%= button_to ‘+1’, up vote_topic_path(topic), method: :post %></td>
解釋：
1.pluralize(topic.votes.count, “vote”)可以藉由判斷前面變數的數量來顯示單數的”vote”或是複數的”votes"
2.button_to ‘+1'是建立一個html按鈕，內文是ʻ+1ʻ（其實你可以在內文加入任何你想要的文字，例如hello、-100等等，whatever you want。
3.vote_topic_path(topic)，則是希望前往的路徑。（這邊不太確定：應該就是前往/topics/:id/upvote(.:format)這個路徑，並執行upvote action。）
4.method: :post，確保使用CRUD中的Create（建立） method，而不是read（讀取）（其實就是html中表格中的post or get method）

註：在create頁面後會出現flash message，在rails裡可以寫成
  format.html { redirect_to @topic, notice: 'Topic was successfully created.' }
而不用像Sinatra一樣手動存在session裡


建立新文章後導回到文章列表
修改以下路徑的@topic
  format.html { redirect_to @topic, notice: 'Topic was successfully created.' } 
  format.json { render :show, status: :created, location: @topic }
改成topics_path
  format.html { redirect_to topics_path, notice: 'Topic was successfully created.’ }
  format.json { render :show, status: :created, location: @topic }
解釋：
format_html，代表網站會回傳HTML內容回去給瀏覽器（或者你可以想format.extension就是在告訴rails遇到不同的檔案類型該如何處理，像是下方的json就表示當檔案類型是json時該怎麼辦。一般來說format預設為html，所以沒有特殊需求可以直接不寫）
redirect_to topic_path，代表執行完後導回某個頁面（這裡是topic_path）
notice: “message”，代表flash message，會顯示在導回的頁面上

註：其實後來發現route可以告訴你很多答案，像是我在show.html.erb的頁面中看到back會前往topics_path（文章列表），所以我只要把原先建立新文章會導往@topic（show detail）的路徑，改成topics_path就可以了

將文章標題變成連結
把以下的code
  <td><%= topic.title %></td>
改成
  <td><%= link_to topic.title, topic %></td>
解釋：
其實就是利用link_to 來設定anchor，topic.title就是anchor text，topic就是href的路徑
註：但為何這裡的路徑是topic還不是很清楚


增加扣分按鈕
這部分我是參考http://railsnote.logdown.com/posts/199641-probe-into-the-railsbridge-rails-bonus-question-marking-button的blog，
在controller的部分加入了downvote這個action
  def downvote
      @topic = Topic.find(parmas[:id]
      @topic.votes.last.destroy
  end
註：目前還不確定TopicsController實際的各項instance method怎麼運作，但是可以想像@topic.votes會列出一個array中的所有值，而last或是first可以挑出其中一個值並執行destroy，藉此達成扣分的按鈕
但是這裡的問題是，投票不能小於零，因為這樣會呈現nil值
