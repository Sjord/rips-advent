# Day 19 - Birch

Can you spot the vulnerability?

```php
class ImageViewer {
    private $file;

    function __construct($file) {
        $this->file = "images/$file";
        $this->createThumbnail();
    }

    function createThumbnail() {
        $e = stripcslashes(
            preg_replace(
                '/[^0-9\\\]/',
                '',
                isset($_GET['size']) ? $_GET['size'] : '25'
            )
        );
        system("/usr/bin/convert {$this->file} --resize $e
                ./thumbs/{$this->file}");
    }

    function __toString() {
        return "<a href={$this->file}>
                <img src=./thumbs/{$this->file}></a>";
    }
}

echo (new ImageViewer("image.png"));
```

[Show Solution](19-birch-solution.md)
