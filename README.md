Celery daemon
===========================

[TOC]

celery 在官方文档中已经说明了如何将一个worker进行daemon，这里结合项目本身的一些情况来说明如何让cmdb中运行的各种worker在后台运行起来.

现在的cmdb已经有了多个worker了，通过监听不同的队列来分别处理不同的后台任务，比如说有一个发送邮件的后台任务，一个定时收取邮件的后台任务，一个自动添加svn权限的后台任务。

之前的做法是通过screen来将这个前台运行的worker来当做后台任务处理，但是这样子有一个问题，就是每次更新了一些相应的代码以后，需要进入到不同的screen中，重新启动对应的worker，对于以后越来越多的worker，显然很不方便。

好在官方有一个daemon的方法，这里说明下如何处理

# Init-script: celeryd

## celeryconfig.py 和tasks.py
`celeryconfig.py`配置好celery
`tasks.py`配置好任务

```
cat /data/www/cmdb/celeryconfig.py

# -*- encoding: utf-8 -*-

CELERY_TIMEZONE = 'Asia/Shanghai'


BROKER_URL = 'amqp://localhost//'
CELERY_IMPORTS = ('tasks', )

CELERY_RESULT_BACKEND = 'amqp'
CELERY_RESULT_PERSISTENT = True
CELERY_TASK_RESULT_EXPIRES = None

CELERY_DEFAULT_QUEUE = 'default'
CELERY_QUEUES = {
    'default': {
        'binding_key': 'task.#',
    },
    'send_mail': {
        'binding_key': 'send_mail.#',
    },
    'recieve_mail': {
        'binding_key': 'recieve_mail.#',
    },
    'workflow_add_server_permission': {
        'binding_key': 'workflow_add_server_permission.#',
    },
    'set_user_host': {
        'binding_key': 'set_user_host.#',
    },
    'add_svn_workflow': {
        'binding_key': 'add_svn_workflow.#',
    },
}
CELERY_DEFAULT_EXCHANGE = 'tasks'
CELERY_DEFAULT_EXCHANGE_TYPE = 'topic'
CELERY_DEFAULT_ROUTING_KEY = 'task.default'
CELERY_ROUTES = {
    'tasks.send_mail': {
        'queue': 'send_mail',
        'routing_key': 'send_mail.a_mail'
    },
    'tasks.recieve_mail': {
        'queue': 'recieve_mail',
        'routing_key': 'recieve_mail.a_mail'
    },
    'tasks.workflow_add_server_permission': {
        'queue': 'workflow_add_server_permission',
        'routing_key': 'workflow_add_server_permission.a_server_permission'
    },
    'tasks.set_user_host': {
        'queue': 'set_user_host',
        'routing_key': 'set_user_host.a_set_user_host'
    },
    'tasks.add_svn_workflow': {
        'queue': 'add_svn_workflow',
        'routing_key': 'add_svn_workflow.a_set_user_host'
    },
}


from datetime import timedelta
from celery.schedules import crontab

CELERYBEAT_SCHEDULE = {
    'recieve_mail': {
        'task': 'tasks.recieve_mail',
        'schedule': timedelta(seconds=60),
    },
    'set_user_host': {
        'task': 'tasks.set_user_host',
        'schedule': crontab(minute=0, hour='*/4'),
    }
}

```


```
cat /data/www/cmdb/tasks.py

# -*- encoding: utf-8 -*-

from celery import Celery

app = Celery()

import celeryconfig

app.config_from_object(celeryconfig)

# from myworkflows.mails import SendEmail
# from myworkflows.mails import RecieveMail


import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

import imapy
from imapy.query_builder import Q

import re

import os
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "cmdb.settings")

import django
django.setup()

from myworkflows.models import WorkflowStateEvent
from myworkflows.utils import do_transition, get_state_user, make_email_notify


from django.contrib.auth.models import User

from cmdb.logs import *
from myworkflows.utils import make_email
from myworkflows.utils import SVN_EXCUTORS
from myworkflows.utils import format_svn
from myworkflows.models import *
from myworkflows.config import *
from assets.models import *
from users.models import UserProfileHost

import requests

ml = MailLog()

from django.db import connections

import json
from datetime import datetime

import traceback


def close_old_connections():
    for conn in connections.all():
        conn.close_if_unusable_or_obsolete()


@app.task(ignore_result=True)
def set_user_host():

    shost_log = SHostLog()

    try:
        # 重新连接数据库
        close_old_connections()

        temporary_permission = UserProfileHost.objects.filter(temporary=1, is_valid=1)

        for t in temporary_permission:
            now = datetime.now()
            if now > t.end_time:
                t.is_valid = 0
                t.save()
        shost_log.logger.info('set host status ok')
    except Exception as e:
        shost_log.logger.error('%s' % (str(e)))


@app.task(ignore_result=True)
def add_svn_workflow(wse_id):
    """自动添加svn权限"""
    wse = WorkflowStateEvent.objects.get(id=wse_id)
    content_object = wse.content_object
    svn_log = SVNLog()

    if isinstance(content_object, SVNWorkflow):
        try:
            # 重新连接数据库
            close_old_connections()

            payload = format_svn(content_object)
            url = 'https://192.168.40.11/api/addprivilege/'
            headers = {
                'Accept': 'application/json',
                'Authorization': 'Token d11205fc792d2d2def44ca55e5252dcbdcea6961',
                'Connection': 'keep-alive',
            }
            r = requests.post(url, headers=headers, data=payload, verify=False)
            svn_log.logger.info('%s: %d: %s' % (content_object.title, r.status_code, r.text))
            msg = r.json()
            result = msg['result']
            data = msg['data']
            if result:
                content_object.status = 0
                svn_log.logger.info('ok')
            else:
                content_object.status = 2
                svn_log.logger.info('%s' % (data))
            content_object.save()
        except requests.exceptions.ConnectionError:
            svn_log.logger.info('time_out')
            content_object.status = 2
            content_object.save()
        except Exception as e:
            svn_log.logger.info('%s' % (str(e)))
            content_object.status = 2
            content_object.save()
```

## celery worker启动脚本
1 创建/etc/init.d/celeryd，内容在[celery repo](https://github.com/celery/celery/blob/master/extra/generic-init.d/celeryd)中

2 修改权限
`chmod 755 /etc/init.d/celeryd`
`chown root:root /etc/init.d/celeryd`


### 配置
具体的配置需要根据你的项目的实际情况，可以参考[doc](http://docs.celeryproject.org/en/latest/userguide/daemonizing.html#init-script-celeryd)

在`/etc/default`目录下，创建celeryd文件

这里列出我的关于执行svn的配置
```
CELERY_BIN="/data/code/cy_devops/bin/celery"

CELERY_APP="tasks:app"

CELERYD_CHDIR="/data/www/cmdb/"

CELERYD_OPTS="--time-limit=300 --concurrency=8 --hostname=1 -Q add_svn_workflow"

CELERYD_LOG_FILE="/var/log/celery/svn.log"

CELERYD_PID_FILE="/var/run/celery/svn.pid"

CELERYD_USER="root"
CELERYD_GROUP="root"

CELERY_CREATE_DIRS=1
```

###  修改启动脚本名称和指定配置文件
从`/etc/init.d/celeryd`的说明中可以看到下面这句话:

```
# Short-Description: celery task worker daemon
### END INIT INFO
#
#
# To implement separate init-scripts, copy this script and give it a different
# name.  That is, if your new application named "little-worker" needs an init,
# you should use:
#
#   cp /etc/init.d/celeryd /etc/init.d/little-worker
#
# You can then configure this by manipulating /etc/default/little-worker.
```

很明显，如果你有多个不同的worker, 你可以分别将/etc/init.d/celeryd和/etc/default/celeryd命名为相同的名称，这样子celeryd会读取同样名称的配置文件，这样子就能很好分别对每个不同的worker进行配置。

以我这里的svn为例，那么/etc/init.d/celeryd重命名为/etc/init.d/svn-worker
/etc/default/celeryd重命名为/etc/default/svn-worker

启动celery worker daemon
`/etc/init.d/svn-worker start`


## 启动 celery beat daemon

**celery beat**按照我的理解，是一个任务调度器，通过这个**celery beat**来读取定时配置文件里面的内容来发送`task`

然后启动另外的`worker`来获取这些定时任务去执行。

拷贝官方的[启动脚本](https://raw.githubusercontent.com/celery/celery/3.1/extra/generic-init.d/celerybeat)到目录`/etc/init.d/`下

在`/etc/default/`目录下，编辑如下配置文件
```
vim /etc/default/celerybeat

CELERY_BIN="/data/code/cy_devops/bin/celery"

CELERY_APP="tasks:app"

CELERYD_CHDIR="/data/www/cmdb/"


CELERYD_LOG_FILE="/var/log/celery/celerybeat.log"

CELERYD_PID_FILE="/var/run/celery/celerybeat.pid"

CELERYD_USER="root"
CELERYD_GROUP="root"

CELERY_CREATE_DIRS=1
```

现在可以启动`celery beat`来发送定时任务了
`/etc/init.d/celerybeat`

### 启动执行定时任务的worker


cmdb还有一个worker，用来定时修改服务器权限的记录

配置启动脚本
`cp /etc/init.d/svn-worker /etc/init.d/shost-beat-worker`

修改配置文件
`cat /etc/default/shost-beat-worker`

```
CELERY_BIN="/data/code/cy_devops/bin/celery"

CELERY_APP="tasks:app"

CELERYD_CHDIR="/data/www/cmdb/"

CELERYD_OPTS="--time-limit=300 --concurrency=8 --hostname=shost-beat-worker -Q set_user_host -B -l debug"

CELERYD_LOG_FILE="/var/log/celery/shost-beat-worker.log"

CELERYD_PID_FILE="/var/run/celery/shost-beat-worker.pid"

CELERYD_USER="root"
CELERYD_GROUP="root"

CELERY_CREATE_DIRS=1
```

启动worker
`/etc/init.d/shost-beat-worker start`
