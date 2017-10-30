---
layout: default
group: howdoi
subgroup: checkout
title: Add custom fields that influence other Checkout fields
menu_title: Add custom fields that influence other Checkout fields
menu_order: 90
version: 2.0
github_link: howdoi/checkout/checkout_custom_checkbox.md
functional_areas:
  - Checkout
---

This topic describes how to add a custom field that influences other fields on the checkout page. 

Let's consider a case where you need to add a check box whose state (selected or cleared) changes the state of other fields: when the check box is selected, the Shipping Address fields get prepopulated with a certain address. To implement such a check box, take the following steps:

1. Create a [plugin]({{page.baseurl}}extension-dev-guide/plugins.html) for the process method of the `<Magento_Checkout_module_dir>/Block/Checkout/LayoutProcessor.php` class.
2. Declare the plugin in your module's `di.xml`.
2. Create a JS component for the check box with custom logic.

## Create a plugin for the `LayoutProcessor`'s process method

In your custom module directory, create the following new file: `<your_module_dir>/Block/Checkout/SomeProcessor.php`. In this file, add the following code sample. This is a plugin that adds a check box, makes the street labels trackable, and assigns dependencies to the check box.

{%highlight php%}
<?php
namespace Magento\Checkout\Block\Checkout;

class SomeProcessor
{
    /**
     * Checkout LayoutProcessor after process plugin.
     *
     * @param \Magento\Checkout\Block\Checkout\LayoutProcessor $processor
     * @param array $jsLayout
     * @return array
     */
    public function afterProcess(\Magento\Checkout\Block\Checkout\LayoutProcessor $processor, $jsLayout)
    {
        $shippingConfiguration = &$jsLayout['components']['checkout']['children']['steps']['children']['shipping-step']
            ['children']['shippingAddress']['children']['shipping-address-fieldset']['children'];
        $billingConfiguration = &$jsLayout['components']['checkout']['children']['steps']['children']['billing-step']
        ['children']['payment']['children']['payments-list']['children'];

        //Checks if shipping step available.
        if (isset($shippingConfiguration)) {
            $shippingConfiguration = $this->processAddress(
                $shippingConfiguration,
                'shippingAddress',
                [
                    'checkoutProvider',
                    'checkout.steps.shipping-step.shippingAddress.shipping-address-fieldset.street',
                    'checkout.steps.shipping-step.shippingAddress.shipping-address-fieldset.city',
                    'checkout.steps.shipping-step.shippingAddress.shipping-address-fieldset.country_id'
                ]
            );
        }

        //Checks if billing step available.
        if (isset($billingConfiguration)) {
            //Iterate over billing forms.
            foreach($billingConfiguration as $key => &$billingForm) {
                //Exclude not billing forms
                if (!strpos($key, '-form')) {
                    continue;
                }

                $billingForm['children']['form-fields']['children'] = $this->processAddress(
                    $billingForm['children']['form-fields']['children'],
                    $billingForm['dataScopePrefix'],
                    [
                        'checkoutProvider',
                        'checkout.steps.billing-step.payment.payments-list.' . $key . '.form-fields.street',
                        'checkout.steps.billing-step.payment.payments-list.' . $key . '.form-fields.city',
                        'checkout.steps.billing-step.payment.payments-list.' . $key . '.form-fields.country_id'
                    ]
                );
            }
        }

        return $jsLayout;
    }

    /**
     * Process provided address to contains checkbox and have trackable address fields.
     *
     * @param $addressFieldset - Address fieldset config.
     * @param $dataScope - data scope
     * @param $deps - list of dependencies
     * @return array
     */
    private function processAddress($addressFieldset, $dataScope, $deps)
    {
        //Creates checkbox.
        $addressFieldset['custom_checkbox'] = [
            'component' => 'Magento_Checkout/js/single-checkbox',
            'config' => [
                'customScope' => $dataScope,
                'template' => 'ui/form/field',
                'prefer' => 'checkbox'
            ],
            'dataScope' => $dataScope . '.custom_checkbox',
            'deps' => $deps,
            'label' => __('Army Address'),
            'provider' => 'checkoutProvider',
            'visible' => true,
            'initialValue' => false,
            'sortOrder' => 10,
            'valueMap' => [
                'true' => true,
                'false' => false
            ]
        ];

        //Makes each address field label trackable.
        if (isset($addressFieldset['street']['children'])) {
            foreach($addressFieldset['street']['children'] as $key => $street) {
                $street['tracks']['label'] = true;
                //Remove .additional class. Can be removed, but style fix provided instead.
                $street['additionalClasses'] = '';
                $addressFieldset['street']['children'][$key] = $street;
            }
        }

        return $addressFieldset;
    }
}
?>
{%endhighlight%}

## Declare plugin in di.xml

In `<your_module_dir>/etc/frontend/di.xml`, declare the plugin you created on the previous step. The plugin name is arbitrary, in our example it's `ProcessAddressConfiguration`:

{% highlight xml%}
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Checkout\Block\Checkout\LayoutProcessor">
        <plugin name="ProcessAddressConfiguration" type="Magento\Checkout\Block\Checkout\SomeProcessor"/>
    </type>
</config>
{%endhighlight%}


## Create a JS component for the check box

In your custom module directory, create the following new file: `<your_module_dir>/view/frontend/web/js/single-checkbox.js`. In this file, add the following code. This is  a JS component that extends `Magento_Ui/js/form/element/single-checkbox.js`. The `onCheckedChanged` method calls the methods that update street labels, change the city and country values, and disable these fields:

{% highlight javascript%}
define([
    'Magento_Ui/js/form/element/single-checkbox',
    'mage/translate'
], function (AbstractField, $t) {
    'use strict';

    return AbstractField.extend({
        defaults: {
            streetLabels: [$t('Company / Section / Unit'), $t('Post Sector Type'), $t('Post Sector')],
            modules: {
                street: '${ $.parentName }.street',
                city: '${ $.parentName }.city',
                country: '${ $.parentName }.country_id'
            }
        },

        updateStreetLabels: function () {
            if (this.value()) {
                this.street().elems.each(function (street, key) {
                    this.street().elems()[key].set('label', this.streetLabels[key]);
                }.bind(this));
            } else {
                this.street().elems.each(function (street, key) {
                    this.street().elems()[key].set('label', '');
                }.bind(this));
            }
        },

        updateCity: function () {
            if (this.value()) {
                this.city().value('Kyiv');
                this.city().disabled(true);
            } else {
                this.city().value('');
                this.city().disabled(false);
            }
        },

        updateCountry: function () {
            if (this.value()) {
                this.country().value('UA');
                this.country().disabled(true);
            } else {
                this.country().value('');
                this.country().disabled(false);
            }
        },

        onCheckedChanged: function () {
            this._super();
            this.updateStreetLabels();
            this.updateCity();
            this.updateCountry();
        }
    });
});
{%endhighlight%}