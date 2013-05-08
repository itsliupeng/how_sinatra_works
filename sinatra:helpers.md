## Sinatra::Helpers ##

Sinatra::Helpers模块定义了一组帮助开发人员更方便的处理用户请求，生成响应页面的方法。这些方法可以在路由，before/after filter和view中使用。

    def status(value=nil)
      response.status = value if value
      response.status
    end

`status`方法用来设置或读取当前响应的状态码。

    def body(value=nil, &block)
      if block_given?
        def block.each; yield(call) end
        response.body = block
      elsif value
        response.body = value
      else
        response.body
      end
    end

`body`方法用来设置或读取当前响应的响应体。`body`方法也可以接受一个block作为参数，但该block并不会被立即执行，而是会给它增加一个`each`方法（Rack标准要求每个响应的body都响应each方法）并保存在response.body中。直到当Rack服务器以each方法读取body内容时，block才会被执行。   


    def redirect(uri, *args)
      if env['HTTP_VERSION'] == 'HTTP/1.1' and env["REQUEST_METHOD"] != 'GET'
        status 303
      else
        status 302
      end

      response['Location'] = uri(uri, settings.absolute_redirects?, settings.prefixed_redirects?)
      halt(*args)
    end

`redirect`方法可以方便的将用户重定向到指定的页面。该方法的第一个参数是将要重定向到的URL，额外的参数会不加修改的传给`halt`方法。只有当HTTP版本为1.1且请求类型不是GET时，状态码会被设定为303([参考HTTP_303](http://en.wikipedia.org/wiki/HTTP_303))，其它情况状态码会被设置为302。`redirect`方法接着会调用`uri`方法将第一个参数处理成标准的URI格式，并将其设置为响应头中的Location字段。最后将额外参数传递给`halt`方法结束处理。   

    def uri(addr = nil, absolute = true, add_script_name = true)
      return addr if addr =~ /\A[A-z][A-z0-9\+\.\-]*:/
      uri = [host = ""]
      if absolute
        host << "http#{'s' if request.secure?}://"
        if request.forwarded? or request.port != (request.secure? ? 443 : 80)
          host << request.host_with_port
        else
          host << request.host
        end
      end
      uri << request.script_name.to_s if add_script_name
      uri << (addr ? addr : request.path_info).to_s
      File.join uri
    end
    
`uri`方法接受一个当前application中的相对路径将其转换成合法的绝对路径。该函数还接受2个可选参数:absolute参数指定要返回绝对路径还是相对路径；add_script_name指定是否在结果中添加script_name信息。这两个可选参数的默认值都是true。   
`uri`方法首先会判断传入的addr参数，如果addr已经是一个包含主机名的绝对路径，则直接返回该参数。接下来会初始化一个空数组用来保存URI中的每个部分。如果absolute参数为true，则将协议，主机名，端口号等信息拼接在一起存入临时数组。接着依次处理script_name和path_info(关于script_name和path_info的区别，请参考我的[另一篇文章](https://gist.github.com/loveky/5185871))。最后用"/"将临时数组中的各部分组合起来，这里使用了File.join方法。

    alias url uri
    alias to uri
    
给`uri`方法创建两个别名`url`和`to`。这是为什么你会看到`redirect to(/xxx)`这样的用法了，其实和`redirect '/xxx'`没有区别，只不过读起code来更通顺。代码的“可读性”，也算是Ruby一直追求的吧。

    def error(code, body=nil)
      code, body    = 500, code.to_str if code.respond_to? :to_str
      response.body = body unless body.nil?
      halt code
    end

`error`方法可以为特定的response code或当有exception出现时增加一个callback(通常是一个block)。     
当code为一个数字时，只有当某个响应的状态码等于code时，callback才会被触发。   
当code为空，也就是不指定code时，当处理请求遇到exception时，callback会被触发。

    def not_found(body=nil)
      error 404, body
    end

`not_found`方法相当于用`error`方法将状态码设置为404。不同之处在于增加了代码的可读性。

    def headers(hash=nil)
      response.headers.merge! hash if hash
      response.headers
    end

`headers`接受一个hash作为参数将其中的键值对设置为响应头中的参数。关于Hash对象的merge!方法参考[ruby-doc](http://ruby-doc.org/core-1.9.3/Hash.html#method-i-merge-21)。

    def session
      request.session
    end

`session`方法返回Sinatra::Request对象中的session方法的返回值，该方法继承自[Rack::Request](http://rack.rubyforge.org/doc/Rack/Request.html#method-i-session)。

    def logger
      request.logger
    end

`logger`方法返回共享的logger对象。

    # Look up a media type by file extension in Rack's mime registry.
    def mime_type(type)
      Base.mime_type(type)
    end

`mime_type`方法接受一个文件扩展名作为参数，并将利用Sinatra::Base类的类方法`mime_type`查询与该扩展名对应的mimetype。

    def content_type(type = nil, params={})
      return response['Content-Type'] unless type
      default = params.delete :default
      mime_type = mime_type(type) || default
      fail "Unknown media type: %p" % type if mime_type.nil?
      mime_type = mime_type.dup
      unless params.include? :charset or settings.add_charset.all? { |p| not p === mime_type }
        params[:charset] = params.delete('charset') || settings.default_encoding
      end
      params.delete :charset if mime_type.include? 'charset'
      unless params.empty?
        mime_type << (mime_type.include?(';') ? ', ' : ';')
        mime_type << params.map { |kv| kv.join('=') }.join(', ')
      end
      response['Content-Type'] = mime_type
    end

`content_type`方法可以接受0个或多个参数，用来读取或者设置响应体的Content-Type属性。  

* 当没有参数传入时，`content_type`方法将返回当前响应头中Content-Type的值。
* 当有一个或两个参数传入时，第一个参数将被传给`mime_type`方法查找对应的mime_type并设置给当前响应头中的Content-Type属性。如果有第二个参数，则假设该参数一个包含mime_type额外属性值(比如字符编码)的hash。  

此外，`content_type`会检查以下2个条件，如果两个条件均不符合，则在Content-Type中添加默认的字符集(Sinatra默认为UTF-8)。   
1.  `content_type`的第二个参数params中已经包含了关于字符集的属性。    
2.  通过type查找到的mime_type不在settings.add_charaset这个配置项中。关于settings.add_charaset配置项参考[Sinatra文档](http://www.sinatrarb.com/intro.html#Available%20Settings)。

在函数的最后，`content_type`会将params中的剩余属性按照key=value;的形式添加到mime_type字符串的末尾并返回。

    # Set the Content-Disposition to "attachment" with the specified filename,
    # instructing the user agents to prompt to save.
    def attachment(filename=nil)
      response['Content-Disposition'] = 'attachment'
      if filename
        params = '; filename="%s"' % File.basename(filename)
        response['Content-Disposition'] << params
        ext = File.extname(filename)
        content_type(ext) unless response['Content-Type'] or ext.empty?
      end
    end

`attachment`方法接收一个可选的文件名作为参数。客户端接到响应后不会直接显示响应内容，而是提示弹出"Save as"提示框，提示用户将响应体以文件的形式保存在本地，文件名就是`attachment`方法接受的参数，如果未指定参数，则这个本地的额文件名根据浏览器不同而不同。这种行为是通过将Content-Disposition设置为attachment实现的。    
此外，`attachment`方法还会检查当前响应体是否设置了Content-Type，如果尚未设置，且参数filename指定的文件名包含扩展名，则通过`content_type`方法查找该扩展名对应的Content-Type并设置给响应体。

    def send_file(path, opts={})
      if opts[:type] or not response['Content-Type']
        content_type opts[:type] || File.extname(path), :default => 'application/octet-stream'
      end

      if opts[:disposition] == 'attachment' || opts[:filename]
        attachment opts[:filename] || path
      elsif opts[:disposition] == 'inline'
        response['Content-Disposition'] = 'inline'
      end

      last_modified opts[:last_modified] if opts[:last_modified]

      file      = Rack::File.new nil
      file.path = path
      result    = file.serving env
      result[1].each { |k,v| headers[k] ||= v }
      halt result[0], result[2]
    rescue Errno::ENOENT
      not_found
    end

`send_file`方法用来方便的将服务器端某个文件的内容作为响应体发送给客户端。该方法接受一个path参数用来指定要发送的文件以及一个可选的opts(一个hash)，用来保存处理请求的额外信息。    
如果在opts中设置了type这个key或者当前响应体尚未设置Content-Type，则`send_file`会尝试为响应体设置Content-Type：    
1. 如果opts中包含type，则根据它的值调用`content_type`方法进行设置。   
2. 如果opts中不包含type，则使用path指定文件名的扩展名作为参数调用`content_type`，同时传入application/octet-stream作为缺省Content-Type。   

如果opts中设置了disposition且值为attachment或者opts中设置了filename，则将调用`attachment`方法提示用户将响应保存成本地文件，否则会将Content-Disposition设置为inline，这意味这响应将会直接显示在浏览器窗口中。   
如果opts中设置了last_modified则使用`last_modified`设置响应头中的Last-Modified。      
最后创建一个Rack::File对象，并通过该对象读取要发送文件的内容并创建响应体。关于Rack::File的详细信息请参考[rubyforge](http://rack.rubyforge.org/doc/classes/Rack/File.html#M000278)。
    
接下来一段code是Stream类的定义。该类用于辅助处理stream。当要以stream响应请求时，响应体将会被设置为一个Stream类的实例。
    
    class Stream
      def self.schedule(*) yield end
      def self.defer(*)    yield end
      
首先定义2个类方法，`schedule`和`defer`。作为调度器(Scheduler)一定要响应的2个方法。Sinatra里用到的两个调度器分别是
EventMachine和Stream类自己。具体使用哪个则取决于环境变量。

      def initialize(scheduler = self.class, keep_open = false, &back)
        @back, @scheduler, @keep_open = back.to_proc, scheduler, keep_open
        @callbacks, @closed = [], false
      end

`initialize`方法接受3个参数，第一个参数是调度器，缺省为Stream。第二个参数是一个布尔值，表示发送数据完成后是否要保留链接open的状态并等待后续数据。第三个参数是一个block。该block会在Rack服务器对响应体调用`each`方法时被执行。    
除了3个参数外，`initialize`方法还会设置2个实例变量。callbacks数据用于保存一些列proc，这些proc会在链接关闭时被执行。closed变量保存了当前链接的状态，默认为false，即链接打开。

      def close
        return if @closed
        @closed = true
        @scheduler.schedule { @callbacks.each { |c| c.call }}
      end
      
`close`会在链接关闭时被调用，作用就是依次执行callbacks数组里保存的所有proc。

      def each(&front)
        @front = front
        @scheduler.defer do
          begin
            @back.call(self)
          rescue Exception => e
            @scheduler.schedule { raise e }
          end
          close unless @keep_open
        end
      end
      
`each`方法被重写，这也是magic发生的地方，决定了stream不同于普通HTTP响应的特性。该方法会将Rack服务器中原本用来向客户端发送数据的代码保存在front变量中，变被动为主动。原本是由Rack依次通过响应体的`each`方法获取数据并发送，现在则成了响应体把发送数据给客户端的代码保存起来，一直等到数据ready后再调用保存的代码发送数据给客户端，实现了异步的数据发送。    
接着`each`方法会执行该Stream实例被创建时的第3个参数中指定的proc。

      def <<(data)
        @scheduler.schedule { @front.call(data.to_s) }
        self
      end
      
`<<`方法用于向客户端发送数据，它会调用`each`方法中保存在front中的代码。

      def callback(&block)
        return yield if @closed
        @callbacks << block
      end

`callback`方法用于向callbacks数组中添加一个成员。如果当前链接已经被关闭，则直接执行这个callback。

      alias errback callback
    end
      
最后，为callback方法创建一个名为errback的方法。至此Stream类的定义结束。


    # Allows to start sending data to the client even though later parts of
    # the response body have not yet been generated.
    #
    # The close parameter specifies whether Stream#close should be called
    # after the block has been executed. This is only relevant for evented
    # servers like Thin or Rainbows.
    def stream(keep_open = false)
      scheduler = env['async.callback'] ? EventMachine : Stream
      current   = @params.dup
      body Stream.new(scheduler, keep_open) { |out| with_params(current) { yield(out) } }
    end
    
`stream`方法用来将当前响应设置为一个stream。首先判断是否设置了async.callback这个环境变量，如果结果为真，则使用EventMachine作为Stream实例的调度器，否则使用Stream类自身。接着生成一个路由参数变量的副本保存在current中。最后创建一个Stream类实例，并将其设置给当前响应的响应体。

    def cache_control(*values)
      if values.last.kind_of?(Hash)
        hash = values.pop
        hash.reject! { |k,v| v == false }
        hash.reject! { |k,v| values << k if v == true }
      else
        hash = {}
      end

      values.map! { |value| value.to_s.tr('_','-') }
      hash.each do |key, value|
        key = key.to_s.tr('_', '-')
        value = value.to_i if key == "max-age"
        values << [key, value].join('=')
      end

      response['Cache-Control'] = values.join(', ') if values.any?
    end

`cache_control`方法用于为响应设置缓存策略，对应的就是HTTP头中的Cache-Control。

    def expires(amount, *values)
      values << {} unless values.last.kind_of?(Hash)

      if amount.is_a? Integer
        time    = Time.now + amount.to_i
        max_age = amount
      else
        time    = time_for amount
        max_age = time - Time.now
      end

      values.last.merge!(:max_age => max_age)
      cache_control(*values)

      response['Expires'] = time.httpdate
    end

`expires`方法用于为响应设置expire时间和缓存策略。该方法可接受若干个参数。第一个参数是一个数字或是一个Time对象。如果是数字则表示响应在多少秒以后将会过期，如果为Time对象，则表示响应在该Time对象对应的时间点后将过期。`expires`方法会计算出响应过期的时间点并将其设置在响应头的Expires字段中。     
此外，`expires`方法会将所有剩余参数传递给`cache_control`方法以设置缓存策略。

    def last_modified(time)
      return unless time
      time = time_for time
      response['Last-Modified'] = time.httpdate
      return if env['HTTP_IF_NONE_MATCH']

      if status == 200 and env['HTTP_IF_MODIFIED_SINCE']
        # compare based on seconds since epoch
        since = Time.httpdate(env['HTTP_IF_MODIFIED_SINCE']).to_i
        halt 304 if since >= time.to_i
      end

      if (success? or status == 412) and env['HTTP_IF_UNMODIFIED_SINCE']
        # compare based on seconds since epoch
        since = Time.httpdate(env['HTTP_IF_UNMODIFIED_SINCE']).to_i
        halt 412 if since < time.to_i
      end
    rescue ArgumentError
    end

`last_modified`方法用于给响应头设置Last-Modified字段。     
如果请求头中设置了If-Modified-Since，且其值晚于参数time对应的时间，则中断请求处理，并设置状态码为304。      
如果请求头中设置了If-Unmodified-Since，且其值早于参数time对应的时间，则中断请求处理，并设置状态码为412。

    def etag(value, options = {})
      # Before touching this code, please double check RFC 2616 14.24 and 14.26.
      options      = {:kind => options} unless Hash === options
      kind         = options[:kind] || :strong
      new_resource = options.fetch(:new_resource) { request.post? }

      unless [:strong, :weak].include?(kind)
        raise ArgumentError, ":strong or :weak expected"
      end

      value = '"%s"' % value
      value = 'W/' + value if kind == :weak
      response['ETag'] = value

      if success? or status == 304
        if etag_matches? env['HTTP_IF_NONE_MATCH'], new_resource
          halt(request.safe? ? 304 : 412)
        end

        if env['HTTP_IF_MATCH']
          halt 412 unless etag_matches? env['HTTP_IF_MATCH'], new_resource
        end
      end
    end

`etag`方法用于为响应头设置ETag字段，并且检查请求头中是否包含If-None-Match，如果包含该字段，且其值包含当前响应头中ETag字段的值，则中断请求处理，并设置响应码为304(HEAD请求或GET操作)或412(其他操作)。

    def back
      request.referer
    end

`back`方法返回请求头中的Referer字段，该字段表明当前请求来源自哪个页面。该方法可以方便开发人员进行重定向，比如`redirect back`。

    def informational?
      status.between? 100, 199
    end
    
`informational?`方法返回状态码是否介于100，199之间。

    def success?
      status.between? 200, 299
    end

`success?`方法返回状态码是否介于200，299之间。

    def redirect?
      status.between? 300, 399
    end

`redirect?`方法返回状态码是否介于300，399之间。

    def client_error?
      status.between? 400, 499
    end

`client_error?`方法返回状态码是否介于400，499之间。

    def server_error?
      status.between? 500, 599
    end

`server_error?`方法返回状态码是否介于500，599之间。

    def not_found?
      status == 404
    end
    
`not_found?`方法返回状态码是否为404。

    def time_for(value)
      if value.respond_to? :to_time
        value.to_time
      elsif value.is_a? Time
        value
      elsif value.respond_to? :new_offset
        # DateTime#to_time does the same on 1.9
        d = value.new_offset 0
        t = Time.utc d.year, d.mon, d.mday, d.hour, d.min, d.sec + d.sec_fraction
        t.getlocal
      elsif value.respond_to? :mday
        # Date#to_time does the same on 1.9
        Time.local(value.year, value.mon, value.mday)
      elsif value.is_a? Numeric
        Time.at value
      else
        Time.parse value.to_s
      end
    rescue ArgumentError => boom
      raise boom
    rescue Exception
      raise ArgumentError, "unable to convert #{value.inspect} to a Time object"
    end
    
`time_for`方法将参数转换为一个Time对象，该方法在`expires`和`last_modified`两个方法中使用。

    private

    # Helper method checking if a ETag value list includes the current ETag.
    def etag_matches?(list, new_resource = request.post?)
      return !new_resource if list == '*'
      list.to_s.split(/\s*,\s*/).include? response['ETag']
    end
    
`etag_matches?`方法接受2个参数，第一个参数list是一个逗号分隔的一个或多个etag组成的字符串，第二个参数new_resource是一个布尔值，表明当前请求是否为POST操作。当list的值为"\*"时，如果new_resource为真，则返回false，否则返回true。当list值不为"\*"时，则返回list包含的一系列etag是否包含当前响应头中ETag字段中的值。

    def with_params(temp_params)
      original, @params = @params, temp_params
      yield
    ensure
      @params = original if original
    end

`with_params`方法接受一个temp_params参数和一个block。首先`with_params`方法会将当前@params中的值保存在original中，并把temp_params赋值给@params变量，接着调用block。不论block是否成功返回，`with_params`都会再次将之前保存的值还给@params。    
调用该方法相当于在给定的环境中执行block中的代码。