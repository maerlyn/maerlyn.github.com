---
layout: post
title: Enum mező Doctrine2-ben
---

A Doctrine2 nagyon szép és jó, viszont a platformfüggetlenség jegyében (helyesen) nem támogatja
beépítve a csak MySQL-ben létező enum mezőt. Számunkra viszont ez elég hasznos volna, lássuk hát,
hogyan tudjuk belevarázsolni Symfony2 keretek közt.

Az osztályok felépítését úgy oldottam meg, hogy van egy ős, amiben le van írva az összes művelet,
valamint igény szerinti számban ennek a gyerekei egyedi névvel és a hozzájuk tartozó értékekkel.

Új típust úgy tudunk létrehozni, hogy a `Doctrine\DBAL\Types\Type` osztályból származtatunk. Íme
az ősünk:

{% highlight php startinline %}
<?php

namespace Foo\BarBundle\DBAL;

use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\Type;

abstract class EnumType extends Type
{
    protected $name;
    protected static $values;

    public function getSqlDeclaration(array $fieldDeclaration,
        AbstractPlatform $platform)
    {
        return
           $platform->getVarcharTypeDeclarationSQL($fieldDeclaration);
    }

    public function convertToPHPValue($value,
        AbstractPlatform $platform)
    {
        return $value;
    }

    public function convertToDatabaseValue($value,
        AbstractPlatform $platform)
    {
        if (!in_array($value, static::$values)) {
            throw new \InvalidArgumentException(
                "Invalid '".$this->name."' value."
            );
        }

        return $value;
    }

    public function getName()
    {
        return $this->name;
    }

    public static function getValues()
    {
        return static::$values;
    }

    public function requiresSQLCommentHint(AbstractPlatform $platform)
    {
        return true;
    }
}
{% endhighlight %}

(A kódba plusz töréseket raktam, hogy kiférjen.)

Részletesen, hogy mi mit csinál:

  - a `getSqlDeclaration` mondja meg, hogyan legyen a mező az adatbázisban deklarálva. Megkapjuk
    hozzá, hogy a mezőnk milyen opciókat kapott, illetve hogy milyen platformon vagyunk épp. Mi
    ezektől függetlenül szöveges mezőt fogunk létrehozni.
    
  - a `convertToPHPValue` alakítja az adatbázisbeli értéket arra, amit a php-ban akarunk látni.
    Mivel közvetlen az értéket tároljuk, ezért egyszerűen visszaadjuk.
    
  - a `convertToDatabaseValue` az előző ellentéte, itt ellenőrizzük azt, hogy érvényes-e az
    egyáltalán.
    
  - a `getName()` adja meg a mezőnk nevét, amit a kommentben használ. Bővebben erről később.
  
  - a `getValues()` csak visszaadja a lehetséges értékeket, erre a formokban lesz szükség.
  
Az utolsó függvényről picit bővebben szeretnék írni. Amikor migrálást csinálunk, akkor a Doctrine az
adatbázis állapota alapján és a kódunk alapján készít egy-egy sémát, aztán ezeket hasonlítja össze.
Az enum mezőnk az elsőben varchar, a másodikban saját típus lesz, ezért minden alkalommal módosítaná,
mert nem tudja megkülönböztetni. E függvényünk igaz visszatérési értéke miatt viszont a mezőhöz
hozzácsap egy "(DC2Type:valamilyenenum)" kommentet, ami alapján később pontosan tudja, milyen
típusúnak is kell lennie.

Konkrét típust két lépésben tudunk létrehozni: származtatunk kell az előző osztályból, valamint
regisztrálni kell azt. A gyerekre egy példa:

{% highlight php startinline %}
<?php

namespace Foo\BarBundle\DBAL;

class FejlecLablecEnumType extends EnumType
{
    protected $name = "fejleclablecenum";
    protected static $values = array(
        "fejléc"    =>  "fejléc",
        "lábléc"    =>  "lábléc",
    );
}
{% endhighlight %}

A regisztráció pedig csak ennyi:

{% highlight php %}
# app/config/config.yml
doctrine:
    dbal:
        types:
            fejleclablecenum: Foo\BarBundle\DBAL\FejlecLablecEnumType
{% endhighlight %}

És máris tudjuk használni az entity-jeink leírásához:

{% highlight php startinline %}
/**
 * @ORM\Column(type="fejleclablecenum")
 */
protected $tipus;
{% endhighlight %}

Illetve a formjainkban:

{% highlight php startinline %}
$tipusok = FejlecLablecEnumType::getValues();
$builder->add("tipus", "choice", array(
    "label"         =>  "Típus",
    "choices"       =>  $tipusok,
    "constraints"   =>  array(
        new Assert\NotBlank(),
        new Assert\Choice(array("choices" => array_keys($tipusok))),
    ),
));
{% endhighlight %}

Ez utóbbi megoldás nem a legszebb, de még nem találtam rá jobbat.

