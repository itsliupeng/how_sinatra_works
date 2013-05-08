## Sinatra::Request ##

Sinatra模块下的Request类继承自Rack::Request，并增加了一些实例方法用于更方便的处理请求。

    def accept
      @env['sinatra.accept'] ||= begin
        entries = @env['HTTP_ACCEPT'].to_s.split(',')
        entries.map { |e| accept_entry(e) }.sort_by(&:last).map(&:first)
      end
    end
    
`accept`方法返回一个基于HTTP请求头中HTTP_ACCEPT参数生成的排序数组，按照[HTTP协议规范](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html)，客户端更prefer的type会排在数组前边。要返回的数组会只有在第一次访问时才会生成并cache在环境变量`sinatra.accept`中。该函数调用了`accept_entry`函数用来根据HTTP_ACCEPT中的每条记录生成一个三元素数组用于排序，关于`accept_entry`会在稍后介绍。

    def preferred_type(*types)
      return accept.first if types.empty?
      types.flatten!
      accept.detect do |pattern|
        type = types.detect { |t| File.fnmatch(pattern, t) }
        return type if type
      end
    end
    
`preferred_type`方法接受0个或多个参数，当没有参数时，将直接返回`accept`返回数组中的第一个元素。如果有一个或多个参数，将会拿`accept`返回数组中的每个pattern逐一与参数列表匹配，直到找到第一个匹配项并返回。如果没有找到匹配项则返回nil。

    alias accept? preferred_type
    alias secure? ssl?

设定两个函数别名。`accetp?`链接到刚刚定义的`preferred_type`。而`secure?`指向继承自Rack::Request的`ssl?`方法，返回当前请求是否是HTTPS请求。

    def forwarded?
      @env.include? "HTTP_X_FORWARDED_HOST"
    end
    
dfadfa 
    
    def safe?
      get? or head? or options? or trace?
    end

`safe?`方法返回当前请求是否会安全（是否会修改服务器上的资源）。

    def idempotent?
      safe? or put? or delete?
    end
    
`idempotent?`方法返回当前请求是否是幂等的（发起多次相同请求跟发起一次请求的结果是否相同）。

    private

    def accept_entry(entry)
      type, *options = entry.delete(' ').split(';')
      quality = 0 # we sort smallest first
      options.delete_if { |e| quality = 1 - e[2..-1].to_f if e.start_with? 'q=' }
      [type, [quality, type.count('*'), 1 - options.size]]
    end

类定义的最后是一个private方法`accept_entry`。这个方法在`accept`方法中用于根据HTTP_ACCEPT字段生成供排序的数组。这里有一点需要注意。在HTTP规范中quality factor默认值是1，并且该值越大表明用户更prefer。而在`accept_entry`方法中正好相反，`quality`缺省值为0，并用1减去HTTP头中的quality factor生成用于排序的quality值。这么做可能是为了配合Ruby中sort默认是升序这一特性吧。