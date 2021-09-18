### Создаем контроллер
1. Создаем `CControllerHostViewMyReport.php` в `zabbix/app/controllers/`
2. Вместо "MyReport" может быть любое имя
3. Копируем исходный код из `zabbix/app/controllers/CControllerHostView.php` 
4. Вставляем скопированный код в `CControllerHostViewMyReport.php`
5. Изменяем имя класса с `CControllerHostView` на `CControllerHostViewMyReport`
### Создаем основной файл для уровня “Представления”
1. Копируем код из файла `zabbix/app/views/monitoring.host.view.php` и сохраняем его в новом файле `zabbix/app/views/monitoring.host.view.my.report.php`
2. Открываем `monitoring.host.view.my.report.php` и заменяем:
    1. `monitoring.host.view.js.php` на `monitoring.host.view.my.report.js.php` (на строчке 29)
    2. `monitoring.host.view.html` на `monitoring.host.view.my.report.html` (на строчке 164)
    3. `setTitle(_('Hosts'))` на `setTitle(_('My Report'))` (на строчке 35)
    4. `->setArgument('action', 'host.view')))` на `->setArgument('action', 'host.view.my.view')))` (на строчке 97)
    5. `->addFormItem((new CVar('action', 'host.view'))->removeId())` на `->addFormItem((new CVar('action', 'host.view.my.report'))->removeId())` (на строчке 100)

### Создаем Javascript файл для уровня “Представления”
1. Копируем код из файла `zabbix/app/views/js/monitoring.host.view.js.php` и сохраняем его в новом файле `zabbix/app/views/js/monitoring.host.view.my.report.js.php`

### Создаем HTML файл для уровня “Представления”
1. Копируем код из файла `zabbix/app/partials/monitoring.host.view.html.php` и сохраняем его в новом файле `zabbix/app/partials/monitoring.host.view.my.report.html.php`
2. Открываем `monitoring.host.view.my.report.html.php` и заменяем:
    1. `->setArgument('action', 'host.view')` на `->setArgument('action', 'host.view.my.report')` (на строчке 25)

### Регистрируем созданные файлы в роутере
1. Открываем `zabbix/include/classes/mvc/CRouter.php`
2. Дублируем строчку 93

```php
'host.view'						=> ['CControllerHostView',							'layout.htmlpage',		'monitoring.host.view'],
```
3. и подставляем имена наших новых файлов

```php
'host.view.my.report'						=> ['CControllerHostViewMyReport',							'layout.htmlpage',		'monitoring.host.view.my.report'],
```

### Добавляем ссылку на новый контролер в меню
1. Открываем `zabbix/include/menu.inc.php`
2. Дублируем строчки 37-41:
```php
(new CMenuItem(_('Hosts')))
    ->setAction('host.view')
    ->setAliases(['web.view', 'charts.view', 'chart2.php', 'chart3.php', 'chart6.php', 'chart7.php',
        'httpdetails.php', 'host_screen.php'
    ]),
```
3. Изменяем `(new CMenuItem(_('Hosts')))` на `(new CMenuItem(_('My Report')))`
4. Изменяем `->setAction('host.view')` на `->setAction('host.view.my.report')`


### Достаем необходимую информацию из базы данных в контролере
1. Открываем CControllerHostViewMyReport.php
2. После строчки 188 ( `$prepared_data = $this->prepareData($filter, $sort, $sortorder);` ) добавляем код (смотри пример ниже), который достанет значение `'Zabbix agent availability'` для каждого хоста
3. Более подробно об HOST API тут - https://www.zabbix.com/documentation/5.0/ru/manual/api/reference/host/get
4. Более подробно об ITEM API тут - https://www.zabbix.com/documentation/5.0/ru/manual/api/reference/item/get
5. Более подробно об HISTORY API тут - https://www.zabbix.com/documentation/5.0/ru/manual/api/reference/history/get
6. Примечание: Ключ `'zabbixAgentAvailability'` будет использоваться в уровне "Представления". Имя коюча может быть каким угодно 

```php
$hosts = $prepared_data['hosts'];

// Get list of hostIds
$hostIds = [];
foreach ($hosts as &$host) {
    $hostIds[$host['hostid']] = true;
}

// Get items from DB
$items = [];
if ($hostIds) {
    $items = API::Item()->get([
        'output' => ['itemid', 'name', 'hostid', 'value_type'],
        'hostids' => array_keys($hostIds),
        'filter' => ['name' => 'Zabbix agent availability'],
        'preservekeys' => true
    ]);
}

// Get list of itemIds
$itemIds = [];
foreach ($items as &$item) {
    $itemIds[$item['itemid']] = true;
}

// GET history from DB
$history = [];
if ($itemIds) {
    $history = API::History()->get([
        'output' => ['itemid', 'clock', 'value'],
        'history' => 3,
        'itemids' => array_keys($itemIds),
        'limit' => 1,
        'sortfield' => 'clock',
        'sortorder' => ZBX_SORT_DOWN
    ]);
}

foreach ($hosts as &$host) {
    $host['zabbixAgentAvailability'] = 'N/A';

    foreach ($items as &$item) {
        if ($item['hostid'] == $host['hostid']) {
            foreach ($history as &$history) {
                if ($history['itemid'] == $item['itemid']) {
                    $host['zabbixAgentAvailability'] = $history['value'];
                }
            }
        }
    }
}
```
7. Добавляем $hosts в $data
```php
$data = [
        'hosts' => $hosts,
        'filter' => $filter,
        'sort' => $sort,
        'sortorder' => $sortorder,
        'refresh_url' => $refresh_curl->getUrl(),
        'refresh_interval' => CWebUser::getRefresh() * 1000,
        'active_tab' => $active_tab
    ] + $prepared_data;
```

### Отображаем найденную информацию в уровне "Представления"
1. Открываем 
2. Заменяем `(new CColHeader(_('Latest data'))),` на `(new CColHeader(_('Zabbix agent availability'))),` на строке 40
4. Удаляем строчки 107-114
```php
[
    new CLink(_('Latest data'),
        (new CUrl('zabbix.php'))
            ->setArgument('action', 'latest.view')
            ->setArgument('filter_set', '1')
            ->setArgument('filter_hostids', [$host['hostid']])
    )
],
```
5. И на их место добавляем новый код

```php
[
    (new CSpan(_($host['zabbixAgentAvailability'])))
],
```

### Отключаем перезагрузку страницы
1. Открываем `CControllerHostViewMyReport.php`
2. Заменяем `'refresh_interval' => CWebUser::getRefresh() * 1000,` на `'refresh_interval' => 0,` на строке 248

или 
2. Заменяем `->setArgument('action', 'host.view.refresh')` на `->setArgument('action', 'host.view.my.report')` на строке 172
