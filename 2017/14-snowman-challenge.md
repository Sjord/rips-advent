# Day 14 - Snowman

Can you spot the vulnerability?

```php
class Carrot {
    const EXTERNAL_DIRECTORY = '/tmp/';
    private $id;
    private $lost = 0;
    private $bought = 0;

    public function __construct($input) {
        $this->id = rand(1, 1000);

        foreach ($input as $field => $count) {
            $this->$field = $count++;
        }
    }

    public function __destruct() {
        file_put_contents(
            self::EXTERNAL_DIRECTORY . $this->id,
            var_export(get_object_vars($this), true)
        );
    }
}

$carrot = new Carrot($_GET);
```

[Show Solution](14-snowman-solution.md)
