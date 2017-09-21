---
layout: post
title:  "Devise Invitable - Manually send invitation token"
date:   2017-09-21 12:00:00 +0100
categories: rails coding devise
---

I recently had the problem, that the mail sent by devise invitable just didn't
reach it's destination, probably due to a spam filter. Nevertheless, that
particular user needed to access the webapp. In earlier versions of rails gem
`devise-invitable` this would not have been a big problem. Just copy the 
invitation_token from the database and append it to the invitation accept
url:

```
https://mycoolapp.com/users/invitations/accept?invitation_token=31enrfjn243r423rv
```

Well with newer versions of devise, this is not working anymore (for obvious reasons) and
the token stored in the database is encrypted.

The only reasonable way I found is to fire up a rais console on the server and
basically resend the invitation to retrieve the raw token.

```
user_to_invite = User.find(email: "peter.muster@test.com")
user_to_invite.deliver_invitation
token = user_to_invite.raw_invitation_token
```

What happens is that deliver_invitation will generate a new invitation token.
Now we can access the raw_invitation_token, which can be used in the accept url.
