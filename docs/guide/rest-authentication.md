Authentication
--------------

Unlike Web applications, RESTful APIs should be stateless, which means sessions or cookies should not
be used. Therefore, each request should come with some sort of authentication credentials because
the user authentication status may not be maintained by sessions or cookies. A common practice is
to send a secret access token with each request to authenticate the user. Since an access token
can be used to uniquely identify and authenticate a user, **the API requests should always be sent
via HTTPS to prevent from man-in-the-middle (MitM) attacks**.

There are different ways to send an access token:

* [HTTP Basic Auth](http://en.wikipedia.org/wiki/Basic_access_authentication): the access token
  is sent as the username. This is should only be used when an access token can be safely stored
  on the API consumer side. For example, the API consumer is a program running on a server.
* Query parameter: the access token is sent as a query parameter in the API URL, e.g.,
  `https://example.com/users?access-token=xxxxxxxx`. Because most Web servers will keep query
  parameters in server logs, this approach should be mainly used to serve `JSONP` requests which
  cannot use HTTP headers to send access tokens.
* [OAuth 2](http://oauth.net/2/): the access token is obtained by the consumer from an authorization
  server and sent to the API server via [HTTP Bearer Tokens](http://tools.ietf.org/html/rfc6750),
  according to the OAuth2 protocol.

Yii supports all of the above authentication methods. You can also easily create new authentication methods.

To enable authentication for your APIs, do the following two steps:

1. Specify which authentication methods you plan to use by configuring the `authenticator` behavior
   in your REST controller classes.
2. Implement [[yii\web\IdentityInterface::findIdentityByAccessToken()]] in your [[yii\web\User::identityClass|user identity class]].


For example, to use HTTP Basic Auth, you may configure `authenticator` as follows,

```php
use yii\helpers\ArrayHelper;
use yii\filters\auth\HttpBasicAuth;

public function behaviors()
{
    return ArrayHelper::merge(parent::behaviors(), [
        'authenticator' => [
            'class' => HttpBasicAuth::className(),
        ],
    ]);
}
```

If you want to support all three authentication methods explained above, you can use `CompositeAuth` like the following,

```php
use yii\helpers\ArrayHelper;
use yii\filters\auth\CompositeAuth;
use yii\filters\auth\HttpBasicAuth;
use yii\filters\auth\HttpBearerAuth;
use yii\filters\auth\QueryParamAuth;

public function behaviors()
{
    return ArrayHelper::merge(parent::behaviors(), [
        'authenticator' => [
            'class' => CompositeAuth::className(),
            'authMethods' => [
                HttpBasicAuth::className(),
                HttpBearerAuth::className(),
                QueryParamAuth::className(),
            ],
        ],
    ]);
}
```

Each element in `authMethods` should be an auth method class name or a configuration array.


Implementation of `findIdentityByAccessToken()` is application specific. For example, in simple scenarios
when each user can only have one access token, you may store the access token in an `access_token` column
in the user table. The method can then be readily implemented in the `User` class as follows,

```php
use yii\db\ActiveRecord;
use yii\web\IdentityInterface;

class User extends ActiveRecord implements IdentityInterface
{
    public static function findIdentityByAccessToken($token, $type = null)
    {
        return static::findOne(['access_token' => $token]);
    }
}
```

After authentication is enabled as described above, for every API request, the requested controller
will try to authenticate the user in its `beforeAction()` step.

If authentication succeeds, the controller will perform other checks (such as rate limiting, authorization)
and then run the action. The authenticated user identity information can be retrieved via `Yii::$app->user->identity`.

If authentication fails, a response with HTTP status 401 will be sent back together with other appropriate headers
(such as a `WWW-Authenticate` header for HTTP Basic Auth).


Authorization
-------------

After a user is authenticated, you probably want to check if he has the permission to perform the requested
action for the requested resource. This process is called *authorization* which is covered in detail in
the [Authorization section](authorization.md).

You may use the Role-Based Access Control (RBAC) component to implementation authorization.

To simplify the authorization check, you may also override the [[yii\rest\Controller::checkAccess()]] method
and then call this method in places where authorization is needed. By default, the built-in actions provided
by [[yii\rest\ActiveController]] will call this method when they are about to run.

```php
/**
 * Checks the privilege of the current user.
 *
 * This method should be overridden to check whether the current user has the privilege
 * to run the specified action against the specified data model.
 * If the user does not have access, a [[ForbiddenHttpException]] should be thrown.
 *
 * @param string $action the ID of the action to be executed
 * @param \yii\base\Model $model the model to be accessed. If null, it means no specific model is being accessed.
 * @param array $params additional parameters
 * @throws ForbiddenHttpException if the user does not have access
 */
public function checkAccess($action, $model = null, $params = [])
{
}
```
