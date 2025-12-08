public static AuthResponse getAccessToken(
        String clientId,
        String clientSecret,
        String username,
        String password,
        String securityToken
    ) {
        Http http = new Http();
        HttpRequest req = new HttpRequest();
        req.setEndpoint('https://login.salesforce.com/services/oauth2/token');
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/x-www-form-urlencoded');
        String body = 'grant_type=password'
            + '&client_id=' + EncodingUtil.urlEncode(clientId, 'UTF-8')
            + '&client_secret=' + EncodingUtil.urlEncode(clientSecret, 'UTF-8')
            + '&username=' + EncodingUtil.urlEncode(username, 'UTF-8')
            + '&password=' + EncodingUtil.urlEncode(password + securityToken, 'UTF-8');
        req.setBody(body);
        HttpResponse res = http.send(req);
        if (res.getStatusCode() == 200) {
            return (AuthResponse)JSON.deserialize(res.getBody(), AuthResponse.class);
        } else {
            throw new CalloutException('Authentication failed: ' + res.getBody());
        }
    }
