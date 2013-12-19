# UseCase your code


## Installation

[![Build Status](https://secure.travis-ci.org/tdantas/usecasing.png)](http://travis-ci.org/tdantas/usecasing)

Add this line to your application's Gemfile:

I did not (YET) upload this code to the rubygems.   
To early adopters use the :git path

    gem 'usecasing', :git => 'https://github.com/tdantas/usecasing.git'

And then execute:

    $ bundle
    
### Usage

Let's build a Invoice System, right ?  
So the product owner will create some usecases/stories to YOU.

Imagine this usecase/story:

````
As a user I want to finalize an Invoice and an email should be delivered to the customer.
````

Let's build a controller

````
	class InvoicesController < ApplicationController
		
		def finalize
		
		    params[:current_user] = current_user
   		    # params = { invoice_id: 123 , current_user: #<User:007> }
			context = FinalizeInvoiceUseCase.perform(params)
			
			if context.success?
				redirect_to invoices_path(context.invoice)
			else
				@errors = context.errors
				redirect_to invoices_path
			end
		
		end
		
	end
````

Ok, What is FinalizeInvoiceUseCase ?

FinalizeInvoiceUseCase will be responsible for perform the Use Case/Story.  
Each usecase should satisfy the [Single Responsability Principle](http://en.wikipedia.org/wiki/Single_responsibility_principle) and to achieve this principle, one usecase depends of others usecases building a Chain of Resposability.


````

	class FinalizeInvoiceUseCase < UseCase::Base
		depends FindInvoice, ValidateToFinalize, FinalizeInvoice, SendEmail
	end	
	
````

IMHO, when I read this Chain I really know what this class will do.   
astute readers will ask: How FindInvoice pass values to ValidateToFinalize ?

When we call in the Controller *FinalizeInvoiceUseCase.perform* we pass a parameter (Hash) to the usecase.

This is what we call context, the usecase context will be shared between all chain.

````
	class FindInvoice < UseCase::Base
		
		def perform
		
			# available because it was added in the controller context 
			user = context.current_user 
			
			# we could do that in one before_filter
			invoice = user.invoices.find(context.invoice_id)
			
			# asign to the context make available to all chain
			context.invoice = invoice
			   
		end

	end
````

Is the invoice valid to be finalized ?

````
	class ValidateToFinalize < UseCase::Base
		
		def perform
			#failure will stop the chain flow and mark the context as error.
			
			failure(:validate, "#{context.invoice.id} not ready to be finalized") unless valid?
		end
		
		private
		def valid?
			#contextual validation to finalize an invoice
		end
	end

````

So, after validate, we already know that the invoice exists and it is ready to be finalized.

````
	class FinalizeInvoice < UseCase::Base
		
		def perform
			invoice = context.invoice
			invoice.finalize! #update database with finalize state
			context.customer = invoice.customer
		end
	
	end
````

Oww, yeah, let's notify the customer

````
	class SendEmail < UseCase::Base
	
		def perform
			to = context.customer.email
			
			# Call whatever service
			EmailService.send('customer_invoice_template', to, locals: { invoice: context.invoice } )
		end
	
	end
````


Let me know what do you think about it.

#### TODO
 
 Create real case examples (40%)



## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
