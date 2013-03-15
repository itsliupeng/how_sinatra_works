##lib\sinatra\main.rb
	require 'sinatra/base'
文件[第1行](https://github.com/sinatra/sinatra/blob/v1.3.5/lib/sinatra/main.rb#L1)首先`require`进来sinatra/base，关于Sinatra所有的magic全部在那里。

	class Application < Base

[第4行](https://github.com/sinatra/sinatra/blob/v1.3.5/lib/sinatra/main.rb#L4)定义一个Application类，继承自Sinatra::Base。

	set :app_file, caller_files.first || $0

[第9行](https://github.com/sinatra/sinatra/blob/v1.3.5/lib/sinatra/main.rb#L9)定义一个全局配置项`:app_file`。当使用classic方式编写sinatra程序时，第一个require 'sinatra'的文件会被当作app\_file。所有其他和路径有关的配置选项都基于这个app\_file的路径设置。
	
	set :run, Proc.new { File.expand_path($0) == File.expand_path(app_file) }

[第11行](https://github.com/sinatra/sinatra/blob/v1.3.5/lib/sinatra/main.rb#L11)定义全局配置项`:run`。这个配置项保存了一个Proc对象，这个Proc中只有一条判断语句，它会判断当前命令行里运行的第一个脚本是否和app\_file是同一文件。换句话说就是判断用户是否显式的调用app\_file。当用户显式调用app\_file时，该配置项为true。只有当该配置项为true时，才会启动内置的web服务器运行我们的sinatra应用。

	 if run? && ARGV.any?

[第13行](https://github.com/sinatra/sinatra/blob/v1.3.5/lib/sinatra/main.rb#L13)判断用户是否给app_file提供了命令行参数，如果`run`为true且有参数，则使用optparse处理参数。

	  at_exit { Application.run! if $!.nil? && Application.run? }

[第25行](https://github.com/sinatra/sinatra/blob/v1.3.5/lib/sinatra/main.rb#L25)注册一个at\_exit事件，当ruby解析完我们的app\_file将要退出时，会触发我们注册的at\_exit事件。如果`run`为true且解析脚本的过程中没有遇到错误，则执行Sinatra::Application的`run!`方法，也就是会启动web服务器啦。

	include Sinatra::Delegator

[第28行](https://github.com/sinatra/sinatra/blob/v1.3.5/lib/sinatra/main.rb#L28)include Sinatra::Delegator模块。该模块会将app\_file中顶层作用域中的get/post/after/before等方法delegate给Sinatra::Application这个类。也正是为什么我们没有定义/get/post这些方法却可以直接使用的原因。我们使用get/post定义路由时，实际上是给Sinatra::Application这个类定义方法。因此当Sinatra::Application.run!启动web服务器时，我们的application就知道如何处理路由了。
