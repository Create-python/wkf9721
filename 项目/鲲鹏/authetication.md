```python
"""
Description: Used to jwt
Copyright: Copyright © Huawei Technologies Co., Ltd. 2019. All rights reserved.
Python version: 3.7
History: 2019-11-25 21:44 created
"""
from django.contrib.auth import get_user_model
from rest_framework import exceptions
from rest_framework.exceptions import ValidationError
from rest_framework.authentication import BaseAuthentication

from .jwt_aut import (JwtAut, _get_jwt_value)


class JSONWebAuthentication(BaseAuthentication):
    """ 认证基本类 """
    www_authenticate_realm = 'api'

    def authenticate(self, request):
        """ 认证用户 """
        jwt_value_user = _get_jwt_value(request)
        if jwt_value_user is None:
            return None
        return JSONWebAuthentication.jwt_auth(jwt_value_user)

    @staticmethod
    def jwt_auth(jwt_value_user):
        """
        jwt认证
        :param jwt_value_user: jwt
        """
        try:
            jwt_obj = JwtAut(jwt_value_user)
        except ValidationError as err:
            raise exceptions.AuthenticationFailed(err.detail)
        user = JSONWebAuthentication._authenticate_credentials(jwt_obj)
        old_token = jwt_obj.get('user_uuid')
        new_token = user.uuid
        if new_token == "" or old_token != new_token:
            msg = 'CrowdedOut'
            raise exceptions.AuthenticationFailed(msg)
        return user, jwt_value_user

    @staticmethod
    def _authenticate_credentials(jwt_obj):
        """ 认证用户，返回用户对象 """
        user_model = get_user_model()
        username = jwt_obj.username
        if not username:
            raise exceptions.AuthenticationFailed('Invalid payload.')
        user_now = user_model.objects.get_by_natural_key(username)
        if not user_now.is_active:
            raise exceptions.AuthenticationFailed('User account is disabled.')
        return user_now

    def authenticate_header(self, request):
        """ 认证jwt头部 """
        return '{0} realm="{1}"'.format('JWT', self.www_authenticate_realm)
```

