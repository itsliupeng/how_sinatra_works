## lib/sinatra/base.rb ##


#### Sinatra::Base类 ####
    def call(env)
      dup.call!(env)
    end

第779-781行定义了`call`方法，该方法是Sinatra与Rack服务器之间的接口。该方法会生成一个当前对象的副本并将以相同参数调用副本的`call!`方法。

    def call!(env) # :nodoc:
      @env      = env
      @request  = Request.new(env)
      @response = Response.new
      @params   = indifferent_params(@request.params)
      template_cache.clear if settings.reload_templates
      force_encoding(@params)

      @response['Content-Type'] = nil
      invoke { dispatch! }
      invoke { error_block!(response.status) }

      unless @response['Content-Type']
        if Array === body and body[0].respond_to? :content_type
          content_type body[0].content_type
        else
          content_type :html
        end
      end

      @response.finish
    end

第785-803行定义了实际处理Rack服务器请求的`call!`方法。该方法只接受一个参数env，它包含了关于请求的所有信息。    
函数开始处首先基于参数env生成一个Request对象，接着生成一个空的Response对象用来保存请求结果。

https://github.com/sinatra/sinatra/blob/master/lib/sinatra/base.rb#L17-L24