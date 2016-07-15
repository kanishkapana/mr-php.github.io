---
layout: post
title: Advanced Multi-Model Forms in Yii2
excerpt: "<p>How to manage multiple child models in a form, supporting validation for each model.</p>"
tags: [yii2, forms]
---

How to manage multiple child models in a form, supporting validation for each model.

Simply click a button to add another child, and it will magically appear on the form.

After submitting the form, any validation errors will be clearly displayed.

![Yii2 Multi-Model Form](https://cloud.githubusercontent.com/assets/51875/16708229/58cdacfc-462a-11e6-96dc-3894f6e1cf6b.jpg)


Consider a `Product` model that has multiple `Parcel` models related, which represent a list of
parcels that belong to the product.  In the form you want to allow adding any number of parcels 
and each one should validate according to the model rules.

## Table Models

Start with the following tables:

```sql
CREATE TABLE `product` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
CREATE TABLE `parcel` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `product_id` int(11) NOT NULL,
  `code` varchar(255) NOT NULL,
  `height` int(11) NOT NULL,
  `width` int(11) NOT NULL,
  `depth` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `product_id` (`product_id`),
  CONSTRAINT `parcel_product_id` FOREIGN KEY (`product_id`) REFERENCES `product` (`id`)
) ENGINE=InnoDB;
```

And the models that represent the tables are:

`models/Product.php`

```php
<?php
namespace app\models;
use \yii\db\ActiveRecord;

class Product extends ActiveRecord
{
    public static function tableName()
    {
        return 'product';
    }
    public function rules()
    {
        return [
            [['name'], 'required'],
            [['name'], 'string', 'max' => 255]
        ];
    }
    public function getParcels()
    {
        return $this->hasMany(Parcel::className(), ['product_id' => 'id']);
    }
}

```

`models/Parcel.php`

```php
<?php
namespace app\models;
use \yii\db\ActiveRecord;

class Parcel extends ActiveRecord
{
    public static function tableName()
    {
        return 'parcel';
    }
    public function rules()
    {
        return [
            [['code', 'height', 'width', 'depth'], 'required'],
            [['product_id', 'height', 'width', 'depth'], 'integer'],
            [['code'], 'string', 'max' => 255],
            [['product_id'], 'exist', 
              'skipOnError' => true, 
              'targetClass' => Product::className(), 
              'targetAttribute' => ['product_id' => 'id']]
        ];
    }
    public function getProduct()
    {
        return $this->hasOne(Product::className(), ['id' => 'product_id']);
    }
}
```

## The Form Model

Now we get to the advanced form, but don't worry, it's all quite straight forward.  We need a model to handle loading, validating and saving the data.  

The purpose of `ProductForm` form is to:
- handle loading `Product` and `Parcel` models
- assigning the submitted data to model attributes
- saving data after validation
- displaying error summaries on the form

`models/form/ProductForm`

```php
<?php
namespace app\models\form;

use app\models\Product;
use app\models\Parcel;
use Yii;
use yii\base\Model;
use yii\widgets\ActiveForm;

class ProductForm extends Model
{
    private $_product;
    private $_parcels;

    public function rules()
    {
        return [
            [['Product'], 'required'],
            [['Parcels'], 'safe'],
        ];
    }

    public function afterValidate()
    {
        $error = false;
        if (!$this->product->validate()) {
            $error = true;
        }
        foreach ($this->parcels as $parcel) {
            if (!$parcel->validate()) {
                $error = true;
            }
        }
        if ($error) {
            $this->addError(null); // add an empty error to prevent saving
        }
        parent::afterValidate();
    }

    public function save()
    {
        if (!$this->validate()) {
            return false;
        }
        $transaction = Yii::$app->db->beginTransaction();
        if (!$this->product->save()) {
            $transaction->rollBack();
            return false;
        }
        foreach ($this->parcels as $parcel) {
            $parcel->product_id = $this->product->id;
            if (!$parcel->save(false)) {
                $transaction->rollBack();
                return false;
            }
        }
        $transaction->commit();
        return true;
    }

    public function getProduct()
    {
        return $this->_product;
    }

    public function setProduct($product)
    {
        if ($product instanceof Product) {
            $this->_product = $product;
        } else if (is_array($product)) {
            $this->_product->setAttributes($product);
        }
    }

    public function getParcels()
    {
        if ($this->_parcels === null) {
            if ($this->product->isNewRecord) {
                $this->_parcels = [];
            } else {
                $this->_parcels = Parcel::find()
                  ->andWhere(['product_id' => $this->product->id])
                  ->all();
            }
        }
        return $this->_parcels;
    }

    public function getParcel($id)
    {
        $parcel = $this->product ? Parcel::find()->where([
            'id' => $id,
            'product_id' => $this->product->id,
        ])->one() : false;
        if (!$parcel) {
            $parcel = new Parcel();
            $parcel->loadDefaultValues();
        }
        return $parcel;
    }

    public function setParcels($parcels)
    {
        unset($parcels['__id__']); // remove the hidden "new Parcel" row
        $this->_parcels = [];
        foreach ($parcels as $id => $parcel) {
            if (is_array($parcel)) {
                $this->_parcels[$id] = $this->getParcel($id);
                $this->_parcels[$id]->setAttributes($parcel);
            } elseif ($parcel instanceof Parcel) {
                $this->_parcels[$id] = $parcel;
            }
        }
    }

    public function errorSummary($form)
    {
        $errorLists = [];
        foreach ($this->getAllModels() as $id => $model) {
            $errorList = $form->errorSummary($model, [
              'header' => '<p>Please fix the following errors for <b>' . $id . '</b></p>',
            ]);
            $errorList = str_replace('<li></li>', '', $errorList); // remove the empty error
            $errorLists[] = $errorList;
        }
        return implode('', $errorLists);
    }

    private function getAllModels()
    {
        $models = [
            'Product' => $this->product,
        ];
        foreach ($this->parcels as $id => $parcel) {
            $models['Parcel.' . $id] = $this->parcels[$id];
        }
        return $models;
    }

}
```

## Controller Actions

The `ProductForm` class is used in `actionUpdate()` and `actionCreate()` methods in `ProductController`.

`controllers/ProductController.php`

```php
<?php
namespace app\controllers;
use app\models\Product;
use Yii;
use yii\web\Controller;

class ProductController extends Controller
{
    public function actionCreate()
    {
        $productForm = new ProductForm();
        $productForm->product = new Product;
        $productForm->setAttributes(Yii::$app->request->post());
        if (Yii::$app->request->post() && $productForm->save()) {
            Yii::$app->getSession()->setFlash('success', 'Product has been created.');
            return $this->redirect(['update', 'id' => $productForm->product->id]);
        } elseif (!Yii::$app->request->isPost) {
            $productForm->load(Yii::$app->request->get());
        }
        return $this->render('create', ['productForm' => $productForm]);
    }
    public function actionUpdate($id)
    {
        $productForm = new ProductForm();
        $productForm->product = $this->findModel($id);
        $productForm->setAttributes(Yii::$app->request->post());
        if (Yii::$app->request->post() && $productForm->save()) {
            Yii::$app->getSession()->setFlash('success', 'Product has been updated.');
            return $this->redirect(['update', 'id' => $productForm->product->id]);
        } elseif (!Yii::$app->request->isPost) {
            $productForm->load(Yii::$app->request->get());
        }
        return $this->render('update', ['productForm' => $productForm]);
    }
    protected function findModel($id)
    {
        if (($model = Product::findOne($id)) !== null) {
            return $model;
        }
        throw new HttpException(404, 'The requested page does not exist.');
    }
}
```

## Form Views

The views `views/product/create.php` and `views/product/update.php` will both render a form.

```php
<?= $this->render('_form', ['productForm' => $productForm]); ?>
```

The form will have a section for the Product, and another section for the Parcels.  

Parcels can be added by clicking the "New Parcel" button, which will copy a hidden form.

When updating a Product, the existing Parcel rows will be displayed.

After saving, each Product and Parcel field will be validated individually, if any fail the error will be displayed at the top as well as on the errored field.

`views/product/_form.php`

```php
<?php
use app\models\Parcel;
use yii\helpers\Html;
use yii\widgets\ActiveForm;

?>
<div class="product-form">

    <?php $form = ActiveForm::begin([
        'enableClientValidation' => false, // TODO get this working with client validation
    ]); ?>

    <?= $productForm->errorSummary($form); ?>

    <fieldset>
        <legend>Product</legend>
        <?= $form->field($productForm->product, 'name')->textInput() ?>
    </fieldset>

    <fieldset>
        <legend>Parcels
            <?php
            // new parcel button
            echo Html::a('New Parcel', 'javascript:void(0);', [
              'id' => 'product-new-parcel-button', 
              'class' => 'pull-right btn btn-default btn-xs'
            ])
            ?>
        </legend>
        <?php
        // parcel table
        $parcel = new Parcel();
        $parcel->loadDefaultValues();
        echo '<table id="product-parcels" class="table table-condensed table-bordered">';
        echo '<thead>';
        echo '<tr>';
        echo '<th>' . $parcel->getAttributeLabel('code') . '</th>';
        echo '<th>' . $parcel->getAttributeLabel('width') . '</th>';
        echo '<th>' . $parcel->getAttributeLabel('height') . '</th>';
        echo '<th>' . $parcel->getAttributeLabel('depth') . '</th>';
        echo '<td>&nbsp;</td>';
        echo '</tr>';
        echo '</thead>';
        echo '</tbody>';
        // existing parcels fields
        foreach ($productForm->parcels as $key => $_parcel) {
          echo '<tr>';
          echo $this->render('_form-product-parcel', [
            'key' => $_parcel->isNewRecord ? (strpos($key, 'new') !== false ? $key : 'new' . $key) : $_parcel->id,
            'form' => $form,
            'parcel' => $_parcel,
          ]);
          echo '</tr>'
        }
        // new parcel fields
        echo '<tr id="product-new-parcel-block" style="display: none;">'
        echo $this->render('_form-product-parcel', [
            'key' => '__id__',
            'form' => $form,
            'parcel' => $parcel,
        ]);
        echo '</tr>'
        echo '</tbody>';
        echo '</table>';
        ?>

        <?php ob_start(); // output buffer the javascript to register later ?>
        <script>
            // add parcel button
            var parcel_k = <?php echo isset($key) ? str_replace('new', '', $key) : 0; ?>;
            $('#product-new-parcel-button').on('click', function () {
                parcel_k += 1;
                $('#product-parcels').find('tbody')
                  .append('<tr>' + $('#product-new-parcel-block').html().replace(/__id__/g, 'new' + parcel_k) + '</tr>');
            });
            // remove parcel button
            $(document).on('click', '.product-remove-parcel-button', function () {
                $(this).closest('tbody tr').remove();
            });
            <?php
            // click add when the form first loads
            if (!Yii::$app->request->isPost && $productForm->product->isNewRecord) 
              echo "$('#product-new-parcel-button').click();";
            ?>
        </script>
        <?php $this->registerJs(str_replace(['<script>', '</script>'], '', ob_get_clean())); ?>

    </fieldset>

    <?= Html::submitButton('Save'); ?>
    <?php ActiveForm::end(); ?>

</div>
```

Inside the above form we render the form fields for the parcels.

`views/product/_form-product-parcel.php`

```php
<?php
use app\models\Parcel;
use yii\helpers\Html;
?>
<td>
    <?= $form->field($parcel, 'code')->textInput([
        'id' => "Parcels_{$key}_code",
        'name' => "Parcels[$key][code]",
    ])->label(false) ?>
</td>
<td>
    <?= $form->field($parcel, 'width')->textInput([
        'id' => "Parcels_{$key}_width",
        'name' => "Parcels[$key][width]",
    ])->label(false) ?>
</td>
<td>
    <?= $form->field($parcel, 'height')->textInput([
        'id' => "Parcels_{$key}_height",
        'name' => "Parcels[$key][height]",
    ])->label(false) ?>
</td>
<td>
    <?= $form->field($parcel, 'depth')->textInput([
        'id' => "Parcels_{$key}_depth",
        'name' => "Parcels[$key][depth]",
    ])->label(false) ?>
</td>
<td>
    <?= Html::a('Remove ' . $key, 'javascript:void(0);', [
      'class' => 'product-remove-parcel-button btn btn-default btn-xs',
    ]) ?>
</td>
```

## Additional Information

The `$_POST` data will look something like the following:

```
$_POST = [
    'Product' => [
        'name' => 'Keyboard and Mouse',
    ],
    'Parcels' => [
        '__id__' => [
            'code' => '',
            'width' => '',
            'height' => '',
            'depth' => '',
        ],
        'new1' => [
            'code' => 'keyboard',
            'width' => '50',
            'height' => '5',
            'depth' => '20',
        ],
        'new2' => [
            'code' => 'mouse',
            'width' => '20',
            'height' => '10',
            'depth' => '20',
        ],
    ],
];
```