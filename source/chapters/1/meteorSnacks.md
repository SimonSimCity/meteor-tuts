---
title: Meteor Tips & Tricks
description: Some interesting tips & tricks that Meteor has to offer.
disqusPage: 'Chapter 1: Meteor Tips & Tricks'
---

## Environment Variables

Meteor uses these variables to know which MongoDB database should it connect to, how it should send emails, and many more:

- MONGO_URL : you don't have to have this by default, but if you connect to another database here is where you would put it
- MAIL_URL : the smtp:// to your email, we'll show you in a bit how easy it is to set it up.
- METEOR_PROFILE : if set to 1, you'll see how much time meteor spends on the building/rebuilding process
- ROOT_URL : the real path of Meteor, default is http://localhost:3000

To specify these variables you should do the following:

```
ROOT_URL="http://localhost:3000" MAIL_URL="smtp://gmail.com:25" meteor run
```

## Run Meteor Easy

Inside your Meteor folder you have a file "package.json", that package keeps track of what npm packages you use, and some other
cool stuff. So for example, if you would want to start an app with diff settings like MAIL_URL, etc, you would do something like this:
```json
{
  ...
  "scripts": {
    "start": "MONGO_URL=mongodb://localhost:27017/meteor-tuts meteor run --settings .deploy/local.json --port 3000",
    "deploy": "We'll get into that in another chapter ;)"
  }
}
```

```
// in your terminal:
npm run start
```

## Meteor.wrapAsync

You will use this to be able to do async operations in your methods. Let's say you use S3, or an npm library, that is not in sync,
 it requires you to specify a callback. So try this:
 
```js
Meteor.methods({
    'something_async'  () {
        coolLibrary.coolFunction((err, res) => {
            // gets here after some time.
        })
    }
})
```

You may have a very weird error saying that code cannot run outside "Fibers". Don't want to get into details on that, but here's how you would do it:

```js
Meteor.methods({
    'something_async': function () {
        const run = Meteor.wrapAsync(coolLibrary.coolFunction, coolLibrary);
        // the second argument is to provide context to the function, 
        // if that function uses "this" inside it, then it will fail without the context specified.
        
        try {
            const results = run("some-argument");
            return results;
            // if the callback accepts err then res (standard), then result will be put in sync into results.
        } catch (e) {
            // if an exception occurs, that exception will be caught here
            // and you can treat it by dispatching a Meteor.Error
        }
    }
})
```

## Timers

You may already be familiar, with setTimeout, setInterval from JavaScript, well, Meteor has them too,
but they will run inside fibers. For example:

```js
Meteor.methods({
    'something_async' () {
        Meteor.setInterval(() => {
            console.log('tick');
        }, 1000);
    }
})
```

After you have called the method, you will get a 'tick' in your console, every 1 second. You will not be able to stop this, 
unless you restart or implement a handler that stops it, so be careful!

## Email

Remember the emails we received in console when we were talking about Users ? Well, a while back, they used this module:
If you don't specify MAIL_URL, all Emails that you send, will go to your console. Pretty cool, right?

If you want an email, we recommend: http://www.mailgun.com/ <- Free for < 10,000 per month

```
// we use %40 to represent @ because the username, because they need to be URI encoded
MAIL_URL="smtp://postmaster%40yourdomain.com:b23f5872166c187ad5b5f1abece071b2@smtp.mailgun.org:25" meteor run
```

```js
// Most Basic Usage
import { Email } from 'meteor/email';

Email.send({
  to: 'you@meteor-tuts.com',
  from: 'no-reply@meteor-tuts.com',
  subject: "I'm an email",
  html: '<p>Hello!</p>'
});
```

Read more: http://docs.meteor.com/api/email.html

## Meteor.defer

Sometimes you want to do something, and then notify the user by email. However, if you use SMTP, sometimes
it takes between 50ms to 1s for the mail to be fully sent, and will give the user the impression that your Meteor app is slow. 
This is why you should use this function:

```js
Meteor.methods({
    'action_plus_email': function () {
        // do something
        
        Meteor.defer(() => {
            Email.send(...)
        })
        
        return 'hello there, user';
    }
})
```

Meteor.defer(fn) is same as Meteor.setTimeout(fn, 0), basically it will do a "background" job in a fiber.

## HTTP

Want to use an external REST API ? No problem for Meteor, it has a super simple HTTP package built-in.

http://docs.meteor.com/api/http.html

```js
Meteor.methods({
    'an_api_call': function () {
        const data = HTTP.get('https://jsonplaceholder.typicode.com/posts/1')
        
        console.log(data);
        
        return data;
    }
})
```

## Assets

http://docs.meteor.com/api/assets.html

Go ahead, put something in "/private/some_folder/test.txt":

```
// meteor shell
Assets.getText('/some_folder/test.txt')
```

You would use this when, for example, you have a business, logic-packed csv, or .xls file.
Or you may have a JSON with car models and makes. 
The idea is that you can have any type of file, even binary, that you can privately use on the server.

## Meteor Settings

https://docs.meteor.com/api/core.html#Meteor-settings

```json
// file: .deploy/local.json
{
    "public": {
        "visible": "Something that the client can see"
    },
    "private": {
        "API_KEY": "XXX"
    }
}
```

You can access the settings from the client-side:
```
Meteor.settings.public.visible
```

You can access all settings from the server-side:
```
Meteor.settings.private.API_KEY
```
