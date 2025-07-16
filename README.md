# iap

import 'dart:async';
import 'package:flutter/foundation.dart';
import 'package:in_app_purchase/in_app_purchase.dart';
import 'package:shared_preferences/shared_preferences.dart';

class IAPManager {
  static final IAPManager _instance = IAPManager._internal();
  factory IAPManager() => _instance;
  IAPManager._internal();

  final InAppPurchase _iap = InAppPurchase.instance;
  late StreamSubscription<List<PurchaseDetails>> _subscription;

  final Set<String> _productIds = {
    'one_time_3_downloads',
    'monthly_motivation',
  };

  List<ProductDetails> products = [];

  Future<void> init() async {
    final available = await _iap.isAvailable();
    if (!available) {
      debugPrint("❌ IAP not available");
      return;
    }

    final response = await _iap.queryProductDetails(_productIds);
    if (response.error == null) {
      products = response.productDetails;
    } else {
      debugPrint("❌ Product query error: ${response.error}");
    }

    _subscription = _iap.purchaseStream.listen((purchaseList) {
      _handlePurchaseUpdates(purchaseList);
    });
  }

  void _handlePurchaseUpdates(List<PurchaseDetails> purchases) async {
    final prefs = await SharedPreferences.getInstance();

    for (final purchase in purchases) {
      if (purchase.status == PurchaseStatus.purchased) {
        if (purchase.productID == 'monthly_subscription') {
          await prefs.setBool('isSubscribed', true);
        } else if (purchase.productID == 'one_time_3_downloads') {
          await prefs.setBool('hasOneTimeAccess', true);
        }
        if (purchase.pendingCompletePurchase) {
          await _iap.completePurchase(purchase);
        }
        debugPrint("✅ Purchase successful: ${purchase.productID}");
      } else if (purchase.status == PurchaseStatus.error) {
        debugPrint("❌ Purchase error: ${purchase.error}");
      }
    }
  }

  Future<void> buyProduct(ProductDetails product) async {
    final param = PurchaseParam(productDetails: product);
    await _iap.buyNonConsumable(purchaseParam: param);
  }

  void dispose() {
    _subscription.cancel();
  }

  Future<bool> isSubscribed() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getBool('isSubscribed') ?? false;
  }

  Future<bool> hasOneTimeAccess() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getBool('hasOneTimeAccess') ?? false;
  }
}
import 'package:flutter/material.dart';
// update with your actual path
import 'package:in_app_purchase/in_app_purchase.dart';
import 'package:motivationlapp/payment/iap_manager.dart';

class IAPTestScreen extends StatefulWidget {
  const IAPTestScreen({super.key});

  @override
  State<IAPTestScreen> createState() => _IAPTestScreenState();
}

class _IAPTestScreenState extends State<IAPTestScreen> {
  final IAPManager _iapManager = IAPManager();

  @override
  void initState() {
    super.initState();
    _iapManager.init().then((_) {
      setState(() {}); // update UI after loading products
    });
  }

  @override
  void dispose() {
    _iapManager.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final products = _iapManager.products;

    return Scaffold(
      appBar: AppBar(title: const Text("IAP Test")),
      body: products.isEmpty
          ? const Center(child: CircularProgressIndicator())
          : ListView.builder(
              itemCount: products.length,
              itemBuilder: (context, index) {
                print("product lent = ${products.length}");
                final product = products[index];
                return ListTile(
                  title: Text(product.title),
                  subtitle: Text(product.description),
                  trailing: ElevatedButton(
                    onPressed: () => _iapManager.buyProduct(product),
                    child: Text("Buy @ ${product.price}"),
                  ),
                );
              },
            ),
    );
  }
}
