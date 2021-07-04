
#### <img src="https://state-of-the-art.org/graphics/kivakit/kivakit-32.png" srcset="https://state-of-the-art.org/graphics/kivakit/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.07.XX - The design of KivaKit validation](#validation)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "validation"></a>

2021.07.XX
 
### The design of KivaKit validation  &nbsp; <img src="https://state-of-the-art.org/graphics/checkmark/checkmark-32.png" srcset="https://state-of-the-art.org/graphics/checkmark/checkmark-32-2x.png 2x" style="vertical-align:baseline"/>

*KivaKit* objects that implement *Validatable* can participate in *validation*. Validation checks an object for consistency and reports any problems. Note that a single object may have more than one *ValidationType*. For example, validation of user input may be somewhat different from the validation of an object about to be inserted into a database:

    public interface Validatable
    {
        Validator validator(ValidationType type);
    }
 
*ValidationType* is an identifier that has sets of classes that should or should not be validated. The *Validator* returned from *Validatable* leverages the *Listener* interface from the KivaKit [*Broadcaster / Listener*](#broadcaster) mini-framework to provide integration with the rest of KivaKit. The *Validator.validate(Listener)* method performs validation, broadcasts any warnings or problems to its *Listener* argument, and returns true if validation discovered no problems.

    public interface Validator
    {
        boolean validate(Listener listener);
    }
 
In the following example, an *Email* object is *Validatable* and its *Validatable.validator()* method returns an anonymous subclass of *BaseValidator*. The *onValidate()* override then provides the actual validation with a series of calls to the *problemIf()* method in *BaseValidator*. For each problem encountered, the validator broadcasts a message. The *BaseValidator* implementation *also* captures these messages, analyzes them and returns true if no problem messages were broadcast by *onValidate()*. This design allows the *Email* class to focus entirely on providing validation logic and not on the plumbing for reporting validation problems.

    public class Email implements Validatable
    {
        Set<EmailAddress> to = new HashSet<>();
        EmailAddress from;
        String subject;
        EmailBody body;
        
        @Override
        public Validator validator(ValidationType type)
        {
            return new BaseValidator()
            {
                @Override
                protected void onValidate()
                {
                    problemIf(to == null, "Recipient (to) is missing");
                    problemIf(from == null, "Sender (from) is missing");
                    problemIf(isEmpty(subject), "Subject is missing or empty");
                    problemIf(body == null, "Body is missing");
                }
            };
        }
    }

The actual details of *BaseValidator* are too complex to fully explore here, but the basic execution flow again is:

    validate(Listener)    // Validate, broadcasting problems to Listener
      onValidate()        // Subclass validation implementation
        problemIf()       // If there is a problem,
          addIfNotNull()  // broadcast it and add it to the list of issues
    issues.isValid()      // Return true if there are no important issues
    
And the implementation of this looks (very roughly) like this:

    public abstract class BaseValidator implements Validator
    {
        // Problems and quibbles captured during validation
        private ValidationIssues issues = new ValidationIssues();
        
        @Override
        public final boolean validate(Listener listener)
        {
            [...]
            
            onValidate();
    
            [...]
            
            return issues.isValid();            
        }

        protected abstract void onValidate();
    
        protected Problem problem(String message, Object... parameters)
        {
            return addIfNotNull(listener.problem(message, parameters));
        }
        
        protected Problem problemIf(boolean invalid, String message, Object... parameters)
        {
            if (invalid)
            {
                return problem(message, parameters);
            }
            return null;
        }

        private <T extends Message> T addIfNotNull(T message)
        {
            issues.add(message);
            return message;
        }
    }

Note that the *validate()* / *onValidate()* methods are an example of the [polymorphic final methods pattern](#polymorphic-final-methods). The validation mini-framework that we've discussed here is available in the kivakit-kernel module in [KivaKit](https://www.kivakit.org).
