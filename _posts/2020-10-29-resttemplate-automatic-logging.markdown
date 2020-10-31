---
layout: post
title:  "RestTemplate Automatic Logging"
date:   2020-10-29 15:57:40 -0600
categories: spring,spring boot,spring mvc,logging
---

A great way to avoid repetitive log statements when logging HTTP requests sent via 
[RestTemplate](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html) 
is to use an [Interceptor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/client/ClientHttpRequestInterceptor.html).
Interceptors give you the ability to access/manipulate the outbound request and the response, as well as take some action before and/or
after the request.

{% highlight java %}
@Slf4j
class LoggingInterceptor implements ClientHttpRequestInterceptor {

    @Override
    public ClientHttpResponse intercept(HttpRequest httpRequest, byte[] bytes,
                                        ClientHttpRequestExecution clientHttpRequestExecution) throws IOException {

        URI uri = httpRequest.getURI();

        log.info("Sending request to {}", uri);

        ClientHttpResponse response = clientHttpRequestExecution.execute(httpRequest, bytes);

        log.info("Received response from {} with status code {}", uri, response.getRawStatusCode());

        return response;
    }

}
{% endhighlight %}

Another common use of Interceptors is to add Authorization headers automatically to outbound HTTP requests. 
Interceptors can be added to RestTemplates like so.

{% highlight java %}

    @Bean
    RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {

        return restTemplateBuilder
                .rootUri("https://www.google.com")
                .interceptors(new LoggingInterceptor())
                .build();
    }
{% endhighlight %}

With the addition of LoggingInterceptor to the RestTemplate, you no longer need to manually add log statements before 
and after every call as they will happen automatically for you. 

But how can you guarantee that all developers will know or remember to do this? To solve this, you can create a 
[BeanPostProcessor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/config/BeanPostProcessor.html). 
As the name implies, a BeanPostProcessor acts on Spring Beans as they are produced by Spring's 
[BeanFactory](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/BeanFactory.html).

{% highlight java %}

@Component
public class RestTemplateBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {

        if (bean instanceof RestTemplate) {

            RestTemplate restTemplate = (RestTemplate) bean;

            restTemplate.getInterceptors().add(new LoggingInterceptor());
        }

        return bean;
    }

}

{% endhighlight %}

With the addition of this BeanPostProcessor, you no longer need to add the LoggingInterceptor to the RestTemplate bean during its
creation in the Config class as seen above. Adding this BeanPostProcessor and LoggingInterceptor to your common library 
is a great way to address this cross-cutting concern, automatically adding the LoggingInterceptor to every RestTemplate defined
 in your applications, and ensuring that every outbound HTTP request is logged.
 
Source code for this entry located [here](https://github.com/matthenry87/blog-code/tree/main/src/main/java/com/matthery87/blogcode/resttemplatelogging).

Credit to Tristan Hanson and Chris Kirk, who's [presentation](https://youtu.be/iWQxTHoSsgY) I attended at Spring One 2019.