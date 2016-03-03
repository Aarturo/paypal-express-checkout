# paypal-express-checkout-simple

[![npm version](https://img.shields.io/npm/v/paypal-express-checkout-dt.svg?style=flat-square)](https://www.npmjs.com/package/paypal-express-checkout-dt)
[![npm downloads](https://img.shields.io/npm/dm/paypal-express-checkout-dt.svg?style=flat-square)](https://www.npmjs.com/package/paypal-express-checkout-dt)
[![dependency Status](https://img.shields.io/david/Aarturo/paypal-express-checkout-dt.svg?style=flat-square)](https://david-dm.org/Aarturo/paypal-express-checkout-dt#info=dependencies)
[![Build Status](https://img.shields.io/travis/Aarturo/paypal-express-checkout-dt.svg?style=flat-square)](https://travis-ci.org/Aarturo/paypal-express-checkout-dt)
[![devDependency Status](https://img.shields.io/david/dev/Aarturo/paypal-express-checkout-dt.svg?style=flat-square)](https://david-dm.org/Aarturo/paypal-express-checkout-dt#info=devDependencies)

PayPal Express Checkout implementation in Node.JS

If you got confused by PayPal instructions, 
If you are looking on how to charge your customers with paypal this is the simple solution.
Whether you are using angularjs or any single page app or you just serve your html pages from node directly you can use this component.  

## Installation

```bash
npm install paypal-express-checkout-dt
```

## Examples

### Simplest example
1. Set your paypal sandbox credentials runnning: `export API_USERNAME=your_username && export API_PASSWORD=your_password && export SIGNATURE=your_signature`
2. Run `cd example && npm i && npm start` and go to http://localhost:8893/index.html
3. you will see the full paypal flow 

## Tests

Run npm test to make sure everything works fine.

## Usage
This is how we implement this component, any improvement is welcoming.

1. SetExpressCheckout: the server gets POST request to setExpressCheckout

```javascript
/**
 * Set the buy parameters.
 * 
 * @param {object}   req
 * @param {object}   res
 * @param {Function} next
 * @return {json}
 */
export function setExpressCheckOut(req, res, next){   
 
  let paypal = PayPal.create(
    config.paypalUser, 
    config.paypalPass, 
    config.paypalSignature, 
    config.paypalSandbox
  )

  paypal.setPayOptions(
    config.paypalBrandName, 
    null, 
    config.paypalLogoUrl
  )
  
  //TODO make a function for this
  //seting billing type as RecurringPayments
  paypal.payOptions["L_BILLINGTYPE0"] = "RecurringPayments"
  paypal.payOptions["L_BILLINGAGREEMENTDESCRIPTION0"] = req.body.product.description

  let paypalItems = {
    name: req.body.product.name,
    description: req.body.product.description,
    quantity: 1,
    amount: req.body.product.amount,
    days: req.body.product.days
  }
  
  paypal.setProducts(paypalItems);
  
  let paymentEnding = moment().add(req.body.product.days, "days")
  let order = uuid.v4()

  paypal.setExpressCheckoutPayment(
    req.body.email, 
    order, 
    req.body.product.amount, 
    req.body.product.description, 
    'USD', 
    config.paypalSuccessUrl, 
    config.paypalCancelUrl, 
    false,
    function(err, data) {
      if (err) {        
        res.status(500).json({success: false, message: err})
      }

      User.findOne({_id: req.body.user}).exec(function (err, userFound){
        if (err) {
          res.status(500).json({success: false, message: err})
        } else {
          let payment = new Payment({
            user: req.body.user,
            starting: new Date(),
                  ending: paymentEnding,
                  fee: req.body.product.amount,
                  type: 1,
                  status: 2,
                  system_payment: {
                    SYSTEM: "paypal",
                    TOKEN: data.token,
                    INVNUM: order
                  }
          })

          payment.save(function (erro, newPayment){
            if (erro) {
              res.status(500).json({success: false, message: erro})
            } else {
              res.redirect(data.redirectUrl)
            }
          })
        }
      })            
    }
  )
}
```

2. With the result from SetExpressCheckOut you can show the result or redirect to GetExpressCheckOutDetails like in this example.

```javascript
/**
 * Charge and get the details.
 * 
 * @param  {object}   req
 * @param  {object}   res
 * @param  {Function} next
 * @return {json}
 */
export function getExpressCheckoutDetails(req, res, next){
  let paypal = PayPal.create(
    config.paypalUser, 
    config.paypalPass, 
    config.paypalSignature, 
    true
  )
  
  paypal.getExpressCheckoutDetails(req.query.token, true, function(err, data) {
    /* update the info in payment system whether is successfully or not */
    data["SYSTEM"] = "Paypal"   
    let status = 0    
    if (data.PAYMENTINFO_0_ACK === 'Success' && data.PAYMENTINFO_0_PAYMENTSTATUS === 'Completed') {
      status = 1
    } else if (data.PAYMENTINFO_0_ACK === 'Success' && data.PAYMENTINFO_0_PAYMENTSTATUS === 'Pending') {
      status = 2      
    }
    Payment.findOneAndUpdate(
      {"system_payment.INVNUM": data.INVNUM},
      {$set: {status: status, system_payment: data}},       
      {new: 1},
      function (erro, updatedPayment) {
        if (erro){
          res.status(500).json({success: false, message: erro, paypal_error: err})
        } else {
          /* check the error from express checkout */
          if (err) {
            res.status(500).json({success: false, message: err})
          } else {
            if (updatedPayment){
              /* check if the payment is compelete */
              let updatedoc = {}
              let _message = ''
              if (data.PAYMENTINFO_0_ACK === 'Success' && data.PAYMENTINFO_0_PAYMENTSTATUS === 'Completed') {               
                updatedoc = {$set: {usertype: 2, payment: updatedPayment._id}}
                _message = "Payment successfuly done"
              } else if (data.PAYMENTINFO_0_ACK === 'Success' && data.PAYMENTINFO_0_PAYMENTSTATUS === 'Pending') {                
                updatedoc = {$set: {payment: updatedPayment._id}}
                _message = "Payment in review"
              }                           
              /* update the user type to vip */
              User.findOneAndUpdate(
                {_id: updatedPayment.user},
                updatedoc,
                {new: 1},
                function (error, updatedUser){
                  if (error) {
                    res.status(500).json({success: false, message: error})
                  } else {
                    if (updatedUser){
                      res.json({success: true, message: _message, data: JSON.stringify(data)})
                    } else {
                      res.status(409).json({success: true, message: "User don't found"})        
                    }
                  }
                }
              )               
            } else {
              res.status(409).json({success: true, message: "Payment don't found", paypal_error: err})
            }
          }
        }
      }
    )
  })
}
``` 

3. How we manage the pending payments. We create a cronjob wich run every 4 hours.

```javascript
/**
 * Check if the pending transaction are now completed
 * @param  {object}   req  [description]
 * @param  {object}   res  [description]
 * @param  {Function} next [description]
 * @return {json}        [description]
 */
export function checkPendingPaypal(req, res, next) {

  let last24hrs = moment().subtract(24,"hours") 
  let query = {created_at: {$gte: new Date(last24hrs.format()), $lte: new Date()}, status: 2}

  Payment.find(query).exec(function (err, foundPayments){
    if (err) {
      res.status(500).json({success: false, message: err})
    } else {
      for (let i in foundPayments) {
        
        let paypal = PayPal.create(
          config.paypalUser, 
          config.paypalPass, 
          config.paypalSignature, 
          true
        )

        if (typeof foundPayments[i].system_payment.PAYMENTINFO_0_TRANSACTIONID !== 'undefined') {
          paypal.getTransactionDetails(foundPayments[i].system_payment.PAYMENTINFO_0_TRANSACTIONID, function(err, data) {           
            if (err) {
              console.log(err)
            } else {
              if (data.PAYMENTSTATUS === 'Completed') {

                Payment.findOneAndUpdate(
                  {"system_payment.PAYMENTINFO_0_TRANSACTIONID": data.TRANSACTIONID},
                  {$set: {status: 1, "system_payment.PAYMENTINFO_0_PAYMENTSTATUS": "Completed", "system_payment.PAYMENTINFO_0_PENDINGREASON": data.PENDINGREASON}},       
                  {new: 1},
                  function (erro, updatedPayment) {
                    if (erro){                  
                      console.log("mongo: " + erro + "paypal: " + err)
                    } else {
                      /* check the error from express checkout */
                      if (err) {                    
                        console.log("paypal: " + err)
                      } else {
                        if (updatedPayment){                                                  
                          /* update the user type to vip */
                          User.findOneAndUpdate(
                            {_id: updatedPayment.user},
                            {$set: {usertype: 2}},
                            {new: 1},
                            function (error, updatedUser){
                              if (error) {                            
                                console.log("mongo: " + error)
                              } else {
                                if (updatedUser){
                                  console.log("Updated successfully payment: " + data.TRANSACTIONID)
                                } else {
                                  console.log("User don't found can't updated his user type")
                                }
                              }
                            }
                          )               
                        } else {                      
                          console.log("Payment don't found can't updated his status")
                        }
                      }
                    }
                })
              }
            }
          })
        }   
      }
      res.json({success: true, message: "Revision pending done"})
    }
  })
} 
```

4. Additionally if you want the add some other pay options do it in this way.

### for instance you can set the billing type.

```javascript
  //old way
  paypal.setPayOptions(
    "BrandName", 
    null, 
    "URLLOGO"
  )
  //new feature to add pay options, you can combine the two
  var options = [
    {name: "L_BILLINGTYPE0", value: "RecurringPayments"},
    {name: "L_BILLINGAGREEMENTDESCRIPTION0", value: "Silly description"}
  ]

  paypal.addPayOption(options);
```
## 
Please feel free to comment and contribute.

## License 
MIT
