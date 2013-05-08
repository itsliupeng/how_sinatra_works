##Sinatra::Response##

Sinatra中的Response类继承自Rack::Response。

    def initialize(*)
      super
      headers['Content-Type'] ||= 'text/html'
    end

`initialize`方法会直接调用父类的`initialize`方法。唯一特殊的地方在于如果header中Content-Type属性为空，则将其设置为tet/html。

    def body=(value)
      value = value.body while Rack::Response === value
      @body = String === value ? [value.to_str] : value
    end

`body`方法作为一个setter方法用来为响应体设置body内容。如果参数是一个Rack::Response对象，则将取参数对象的body属性用来赋值。如果参数一个字符串对象则构建一个只包含改对象的字符串数组并赋值，否则将直接用参数进行赋值。

    def each
      block_given? ? super : enum_for(:each)
    end

`each`方法会判断当前调用是否提供了一个block。如果有提供block则直接调用父类的对应方法。否则基于当前对象生成一个Enumerator。

    def finish
      result = body

      if drop_content_info?
        headers.delete "Content-Length"
        headers.delete "Content-Type"
      end

      if drop_body?
        close
        result = []
      end

      if calculate_content_length?
        # if some other code has already set Content-Length, don't muck with it
        # currently, this would be the static file-handler
        headers["Content-Length"] = body.inject(0) { |l, p| l + Rack::Utils.bytesize(p) }.to_s
      end

      [status.to_i, header, result]
    end

`finish`用于生成一个标准的Rack响应。对父类Rack::Response中的`finish`方法做了扩展。  
如果响应的状态码为1xx或着没有响应体则将响应头中的Content-Length和Content-Type字段删除。   
如果状态码表明该请求无响应体则将body设置为空数组。  
接着判断是否需要为响应设置Content-Length头。
最后按照Rack标准生成一个包含3个元素（依次为状态码，响应头，响应体）的数组。

接下来是几个辅助其他方法的pravite方法：

    def calculate_content_length?
      headers["Content-Type"] and not headers["Content-Length"] and Array === body
    end

`calculate_content_length?`用于判断是否要自动为响应设置Content-Length头。只有当响应头中设置了Content-Type但并未设置Content-Length且响应体是一个数组时返回真。

    def drop_content_info?
      status.to_i / 100 == 1 or drop_body?
    end

当状态码为1xx或响应不包含响应体时返回真。

    def drop_body?
      [204, 205, 304].include?(status.to_i)
    end

当状态码为204，205，304中的某一个时为真。关于HTTP状态码可参考[wikipedia](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes)或[HTTP状态码详解](http://www.ostools.net/commons?type=5)。