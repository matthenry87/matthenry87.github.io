---
layout: post
title:  "@NotEmptyIfPresent Custom Validation Annotation"
date:   2020-10-31 15:57:40 -0600
categories: spring,spring boot,spring mvc
---

In some scenarios, you may have query parameters or request model fields that are optional. When these optional fields are
 provided, we often want to make sure the field is not empty. Existing annotations such as @NotNull, @NotEmpty, and 
 @NotBlank won't work for an optional field as they don't have the conditional aspect that we're looking for.

In addition, in the case of strings and lists, once this variable hits the service layer we typically have to check it 
for null as well as checking to see if the variable is empty. These checks cause us to need more conditional statements
in our validation logic, which in turn means we require more unit tests to cover them.

In order to achieve this: ![notemptyifpresent-conditional-coverage.png](/assets/images/notemptyifpresent-conditional-coverage.png)

3 unit tests ar required:

{% highlight java %}

    @Test
    void doSomething() {

        stringService.doSomething(null);
    }

    @Test
    void doSomething2() {

        stringService.doSomething("");
    }

    @Test
    void doSomething3() {

        stringService.doSomething("foo");
    }

{% endhighlight %}

To eliminate some of this conditional logic, and the tests that come along with it, we can use a custom validation 
annotation which will plug right into the existing Spring validation framework. 

The POM depedencies required.

{% highlight xml %}

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.hibernate.validator</groupId>
            <artifactId>hibernate-validator</artifactId>
            <version>6.1.6.Final</version>
        </dependency>

{% endhighlight %}

The annotation.

{% highlight java %}

@Documented
@Constraint(validatedBy = NotEmptyIfPresentValidator.class)
@Target({ ElementType.FIELD, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
public @interface NotEmptyIfPresent {

    String message() default "Must not be empty if present";

    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};

}

{% endhighlight %}

We also need the Validator class that will do the actual validation.

{% highlight java %}

public class NotEmptyIfPresentValidator implements ConstraintValidator<NotEmptyIfPresent, Object> {

    @Override
    public boolean isValid(Object object, ConstraintValidatorContext constraintValidatorContext) {

        if (object == null) {

            return true;
        }

        if (object instanceof List) {

            List<?> list = (List<?>) object;

            return !list.isEmpty();
        }

        if (object instanceof String) {

            String string = (String) object;

            return !string.trim().isEmpty();
        }

        return true;
    }

}

{% endhighlight %}

Now this annotation can be used on both query parameters and fields on request objects to cause validation errors.

{% highlight java %}

@Validated
@RestController
public class Controller {

    @GetMapping
    public Map<String, String> get(@NotEmptyIfPresent @RequestParam(required = false) String string,
                                   @NotEmptyIfPresent @RequestParam(required = false) List<String> strings) {

        HashMap<String, String> map = new HashMap<>();

        map.put("string", string);

        return map;
    }

    @PostMapping
    public Map<String, String> post(@Validated @RequestBody RequestObject requestObject) {

        return new HashMap<>();
    }

    @Getter
    @Setter
    private static class RequestObject {

        @NotEmptyIfPresent
        private String string;
    }
}

{% endhighlight %}

With this annotation is in place, we know that these optional variables are either going to be null or have an actual
non-empty value so only a null check is required in the business logic. Don't forget to create ExceptionHandlers for the 
resulting exceptions so that a 400 response gets returned instead of a 500.

Source code for this entry located [here](https://github.com/matthenry87/blog-code/tree/main/src/main/java/com/matthery87/blogcode/notemptyifpresent).
