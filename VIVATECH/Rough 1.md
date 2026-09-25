
```yml
spring:  
  datasource:  
    url: "jdbc:mariadb://127.0.0.1:3306/crofu_telesom"  
    username: "root"  
    password: "Admin@123"  
  jpa:  
    open-in-view: false  
    hibernate:  
      ddl-auto: update  
  web:  
    resources:  
      static-locations: file:///home/core/crofu/diagrams  
  mvc:  
    static-path-pattern: /static/**  
springdoc:  
  packagesToScan: net.innovii.crofu.comp.ctrl  
  pathsToMatch: /secure/**, /public/**   
server:  
  address: localhost  
  port: 8073  
  servlet:  
    context-path: /crowd-funding  
jwt:  
  key:  
    private: crofu-delivery/src/main/resources/config/jwt.private.key  
    public: crofu-delivery/src/main/resources/config/jwt.public.key  
service:  
  name: crofu  
payment:  
  type: WAAFI  
  gw:  
    url: "http://149.28.148.198:8073/crowd-funding/public/payment/pay/sdf"  
    user:  
      name: pgw  
      pass:  pgw  
  my:  
    callback:  
      url: "http://149.28.148.198:8073/crowd-funding/public/payment/status"  
  waafi:  
    url: https://api.waafipay.net/asm  
sms:  
  testing: true  
  gw:  
    url: "https://chatbotapi.telesom.com/sdf/web/sms/forward"  
    user:  
      name: mhealthSMS  
      pass: "aF3$jUH#3t7h35H"  
  kannel:  
    first: false  
    from: OTP  
    charset: utf-8  
    user:  
      name: tester  
      pass: foobar  
    uri:  
      scheme: http  
      host: 172.16.53.115  
      port: 13013  
      path: /cgi-bin/sendsms  
otp:  
  from:  
    number: 400  
  register:  
    message: "Crowd Funding registration code: %s (valid for 1h)."  
    message-subject: "CroFu Registration"  
  login:  
    message: "Crowd Funding login code: %s"  
    message-subject: "CroFu Login"  
  reset:  
    message: "Crowd Funding password reset code: %s"  
    message-subject: "CroFu Password Reset"  
  invite:  
    message:  
      sms: "CroFu Invite Code: %s"  
      email: "Hi %s,\nAn admin created a %s account for you. Click the link below to set your password (valid for 24 hours):\n%s\nIf you didn’t expect this, please ignore this email."  
      email-subject: "Set up your  Crowdfund account"  
      front: "http://149.28.148.198:7070/crowdfunding/#/authset-password/?token="  
  test:  
    admin:  
      special-value: 555555  
    campaign-manager:  
      special-value: 444444  
      login: aidaruscm, saqzia  
    regular-user:  
      special-value: 333333  
      login: mdzia  
media:  
  store:  
    base-dir: /home/core/crofu/  
    dir:  
      user: media/user  
      campaign: media/campaign  
      campaign-draft: media/campaign-draft  
      category: media/category  
super-admin-direct-login-enabled: true
```