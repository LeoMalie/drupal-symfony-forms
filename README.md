# drupal-symfony-forms
Drupal 10 feat Symfony Forms 7 feat Twig 3

The goal here is to use Symfony Forms (including Twig related features) inside a Drupal custom module. \
DO NOT USE IN PRODUCTION! This README is for adventurers and the solution is definately not production ready. \
Help welcome regarding FormFactoryInterface DI issue and the messy Twig part.  

Starting with :
- drupal/core-recommended : 10.4.7
- twig/twig : v3.19

First install symfony/form and symfony/twig-bridge :
```shell
composer require symfony/form symfony/twig-bridge -W
```
=> You should get 7.* for both. It seems to be possible with Drupal 9 (Symfony 4-5) but a bit more difficult.

In your custom module, create your Symfony Form :
```php
# my_module/src/Form/MyFormType.php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Form;

use Drupal\Core\StringTranslation\StringTranslationTrait;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class MyFormType extends AbstractType
{
    use StringTranslationTrait;

    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('foo', TextType::class, [
                'label' => $this->t('Foo'),
                'required' => false,
            ])
        ;
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => null,
        ]);
    }
}
```

In your controller you can now use your form and pass it to the :
```php
# my_module/src/Controller/MyController.php

public function index(): array
{
     $formSearch = $this->createForm(MyFormType::class);
     $formSearch->handleRequest();
     if ($formSearch->isSubmitted() && $formSearch->isValid()) {
         $array = $formSearch->getData();
         // ...
     }
     return [
        'index' => [
            '#formSearch' => $formSearch->createView(), // remember to add this key in my_module.module
        ],
        // ...
     ]; 
}
```

Now create the createForm method :
```php
# my_module/src/Controller/MyController.php

private function createForm(string $formClass): FormInterface
{
    // Get Twig Environment service
    $twig = Drupal::getContainer()?->get('twig');
    if (!$twig instanceof Environment) {
        throw new RuntimeException();
    }

    // Create a new engine using our theme
    $formEngine = new TwigRendererEngine([
        // Note : 3rd arg of "t" function will mismatch between Symfony theme and Drupal
        // We will need a patch to solve this
        'my-module-theme.html.twig',
    ], $twig);
    // Give to loaders additional paths for the theme 
    /** @var ChainLoader $chainLoader */
    $chainLoader = $twig->getLoader();
    foreach($chainLoader->getLoaders() as $loader) {
        if ($loader instanceof FilesystemLoader) {
            // Add the theme path to the loader
            $loader->addPath('modules/custom/my_module/templates');
            // Same operation for the parent theme
            $loader->addPath('../vendor/symfony/twig-bridge/Resources/views/Form');
        }
    }
    // Tell Twig how to handle Symfony Forms
    $twig->addRuntimeLoader(new FactoryRuntimeLoader([
        FormRenderer::class => function () use ($formEngine) {
            return new FormRenderer($formEngine);
        }
    ]));
    // Add Symfony Forms extension (form_start, form_end, ...)
    $twig->addExtension(new FormExtension());

    // Unable to use DI to use a Symfony FormFactoryInterface => Help welcome
    $formFactory = new FormFactory(new FormRegistry([], new ResolvedFormTypeFactory()));
    return $formFactory->create($formClass);
}
```

Next step is to patch Drupal t function to accept null as $options (you can use a patch composer)
```php
# core/includes/bootstrap.inc
function t($string, array $args = [], ?array $options = []) {
  return new TranslatableMarkup((string) $string, $args, $options ?? []);
}
```

Create the theme that will help you to overide widgets :
```twig
# my_module/templates/my-module-theme.html.twig
{% use "bootstrap_4_layout.html.twig" %}
{# @see vendor/symfony/twig-bridge/Resources/views/Form/bootstrap_4_base_layout.html.twig #}
{# https://symfony.com/doc/2.x/reference/forms/twig_reference.html #}
```

Final step the template associated to the controller route :
```twig
# my_module/templates/my-module-index.html.twig
{{ form_start(formSearch) }}
{{ form_row(formSearch.foo) }}
{{ form_end(formSearch) }}
```


