
#  L-MobileSales Mini 

1. Authentication Module
// Login Screen Structure
class LoginScreen extends HookWidget {
  @override
  Widget build(BuildContext context) {
    final formKey = useMemoized(() => GlobalKey<FormState>());
    final authNotifier = useProvider(authProvider.notifier);
    
    return Scaffold(
      appBar: AppBar(title: Text('Leysco Login')),
      body: SingleChildScrollView(
        child: Padding(
          padding: EdgeInsets.all(16.0),
          child: Form(
            key: formKey,
            child: Column(
              children: [
                BrandLogo(),
                EmailField(validator: validateEmail),
                PasswordField(validator: validatePassword),
                RememberMeCheckbox(),
                ForgotPasswordLink(),
                LoginButton(
                  onPressed: () async {
                    if (formKey.currentState!.validate()) {
                      await authNotifier.login(
                        email: emailController.text,
                        password: passwordController.text,
                        rememberMe: rememberMe.value
                      );
                    }
                  }
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}

Password Validation
String? validatePassword(String? value) {
  if (value == null || value.isEmpty) return 'Required';
  if (value.length < 8) return 'Minimum 8 characters';
  if (!value.contains(RegExp(r'[A-Z]'))) return 'Need 1 uppercase letter';
  if (!value.contains(RegExp(r'[0-9]'))) return 'Need 1 number';
  if (!value.contains(RegExp(r'[!@#$%^&*(),.?":{}|<>]'))) {
    return 'Need 1 special character';
  }
  return null;
}

2. Dashboard & Home Screen
// Dashboard Structure
class DashboardScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, ScopedReader watch) {
    final salesData = watch(salesProvider);
    final inventoryStatus = watch(inventoryProvider);
    
    return RefreshIndicator(
      onRefresh: () async {
        await context.read(salesProvider.notifier).refresh();
        await context.read(inventoryProvider.notifier).refresh();
      },
      child: SingleChildScrollView(
        child: Column(
          children: [
            DashboardHeader(),
            if (salesData.isLoading) ShimmerLoader(),
            if (salesData.hasValue) SalesSummaryCard(salesData.value!),
            QuickActionsGrid(),
            if (inventoryStatus.isLoading) ShimmerLoader(),
            if (inventoryStatus.hasValue) InventoryStatusCard(inventoryStatus.value!),
            RecentActivityList(),
          ],
        ),
      ),
    );
  }
}

3. Inventory Management
// Product List with Pagination
class ProductListScreen extends StatefulWidget {
  @override
  _ProductListScreenState createState() => _ProductListScreenState();
}

class _ProductListScreenState extends State<ProductListScreen> {
  final PagingController<int, Product> _pagingController = PagingController(firstPageKey: 0);

  @override
  void initState() {
    _pagingController.addPageRequestListener((pageKey) {
      _fetchPage(pageKey);
    });
    super.initState();
  }

  Future<void> _fetchPage(int pageKey) async {
    try {
      final newItems = await context.read(productProvider).getProducts(pageKey);
      final isLastPage = newItems.length < 20;
      if (isLastPage) {
        _pagingController.appendLastPage(newItems);
      } else {
        final nextPageKey = pageKey + newItems.length;
        _pagingController.appendPage(newItems, nextPageKey);
      }
    } catch (error) {
      _pagingController.error = error;
    }
  }

  @override
  Widget build(BuildContext context) {
    return PagedListView<int, Product>(
      pagingController: _pagingController,
      builderDelegate: PagedChildBuilderDelegate<Product>(
        itemBuilder: (context, product, index) => ProductListItem(product),
      ),
    );
  }
}

4. Sales Transaction Module
// Sales Order Creation Flow
class SalesOrderScreen extends ConsumerWidget {
  final _pageController = PageController();
  
  @override
  Widget build(BuildContext context, ScopedReader watch) {
    final salesState = watch(salesOrderProvider);
    
    return Scaffold(
      appBar: AppBar(title: Text('New Sales Order')),
      body: Column(
        children: [
          SalesProgressStepper(currentStep: salesState.currentStep),
          Expanded(
            child: PageView(
              controller: _pageController,
              physics: NeverScrollableScrollPhysics(),
              children: [
                CustomerSelectionStep(),
                ProductSelectionStep(),
                QuantityPricingStep(),
                OrderSummaryStep(),
                OrderConfirmationStep(),
              ],
            ),
          ),
          NavigationButtons(
            currentStep: salesState.currentStep,
            onNext: () => _pageController.nextPage(
              duration: Duration(milliseconds: 300),
              curve: Curves.easeInOut
            ),
            onBack: () => _pageController.previousPage(
              duration: Duration(milliseconds: 300),
              curve: Curves.easeInOut
            ),
          ),
        ],
      ),
    );
  }
}

5. Customer Management
// Customer Map View
class CustomerMapScreen extends StatefulWidget {
  @override
  _CustomerMapScreenState createState() => _CustomerMapScreenState();
}

class _CustomerMapScreenState extends State<CustomerMapScreen> {
  late GoogleMapController mapController;
  final Set<Marker> _markers = {};

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Customer Locations')),
      body: GoogleMap(
        onMapCreated: (controller) {
          mapController = controller;
          _loadCustomerMarkers();
        },
        initialCameraPosition: CameraPosition(
          target: LatLng(0, 0),
          zoom: 10,
        ),
        markers: _markers,
      ),
    );
  }

  Future<void> _loadCustomerMarkers() async {
    final customers = await context.read(customerProvider).getCustomers();
    setState(() {
      _markers.addAll(customers.map((customer) => Marker(
        markerId: MarkerId(customer.id),
        position: LatLng(customer.lat, customer.lng),
        infoWindow: InfoWindow(title: customer.name),
        icon: _getMarkerIcon(customer.category),
      )));
    });
  }
}

6. Notifications System
// Notification Toast System
class NotificationToast {
  static void show({
    required BuildContext context,
    required String message,
    required NotificationType type,
    Duration duration = const Duration(seconds: 3),
  }) {
    final overlay = Overlay.of(context);
    final overlayEntry = OverlayEntry(
      builder: (context) => Positioned(
        top: MediaQuery.of(context).padding.top + 10,
        left: 10,
        right: 10,
        child: Material(
          color: Colors.transparent,
          child: AnimatedToast(
            message: message,
            type: type,
          ),
        ),
      ),
    );

    overlay.insert(overlayEntry);
    Future.delayed(duration, overlayEntry.remove);
  }
}

class AnimatedToast extends StatefulWidget {
  final String message;
  final NotificationType type;
  
  const AnimatedToast({required this.message, required this.type});

  @override
  _AnimatedToastState createState() => _AnimatedToastState();
}

class _AnimatedToastState extends State<AnimatedToast> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<Offset> _slideAnimation;
  late Animation<double> _opacityAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: const Duration(milliseconds: 300),
      vsync: this,
    );
    
    _slideAnimation = Tween<Offset>(
      begin: Offset(0, -1),
      end: Offset(0, 0),
    ).animate(CurvedAnimation(
      parent: _controller,
      curve: Curves.easeOut,
    ));
    
    _opacityAnimation = Tween<double>(
      begin: 0,
      end: 1,
    ).animate(CurvedAnimation(
      parent: _controller,
      curve: Curves.easeIn,
    ));
    
    _controller.forward();
  }

  @override
  Widget build(BuildContext context) {
    return SlideTransition(
      position: _slideAnimation,
      child: FadeTransition(
        opacity: _opacityAnimation,
        child: Container(
          padding: EdgeInsets.all(16),
          decoration: BoxDecoration(
            color: _getBackgroundColor(),
            borderRadius: BorderRadius.circular(8),
            boxShadow: [
              BoxShadow(
                color: Colors.black26,
                blurRadius: 10,
                offset: Offset(0, 4),
              ),
            ],
          ),
          child: Row(
            children: [
              Icon(_getIcon(), color: Colors.white),
              SizedBox(width: 12),
              Expanded(child: Text(widget.message, style: TextStyle(color: Colors.white))),
              IconButton(
                icon: Icon(Icons.close, color: Colors.white, size: 20),
                onPressed: () => _controller.reverse().then((_) => Navigator.of(context).pop()),
              ),
            ],
          ),
        ),
      ),
    );
  }
  
  Color _getBackgroundColor() {
    switch(widget.type) {
      case NotificationType.success: return Colors.green;
      case NotificationType.error: return Colors.red;
      case NotificationType.warning: return Colors.orange;
      case NotificationType.info: return Colors.blue;
    }
  }
  
  IconData _getIcon() {
    switch(widget.type) {
      case NotificationType.success: return Icons.check_circle;
      case NotificationType.error: return Icons.error;
      case NotificationType.warning: return Icons.warning;
      case NotificationType.info: return Icons.info;
    }
  }
}

Project Structure Recommendation
lib/
├── core/
│   ├── constants/
│   ├── utils/
│   ├── widgets/
│   └── styles/
├── data/
│   ├── models/
│   ├── repositories/
│   └── datasources/
├── features/
│   ├── auth/
│   │   ├── presentation/
│   │   │   ├── screens/
│   │   │   ├── widgets/
│   │   │   └── auth_flow.dart
│   │   └── application/
│   │       ├── notifiers/
│   │       └── providers/
│   ├── dashboard/
│   ├── inventory/
│   ├── sales/
│   ├── customers/
│   └── notifications/
└── main.dart


MOCK DATA AND RESOURCES 
1. User Data (user_data.json)
[
  {
    "id": "USR-001",
    "username": "LEYS-1001",
    "first_name": "David",
    "last_name": "Kariuki",
    "email": "david.kariuki@leysco.co.ke",
    "role": "Sales Manager",
    "permissions": ["view_sales", "create_sales"]
  },
  {
    "id": "USR-002",
    "username": "LEYS-1002",
    "first_name": "Jane",
    "last_name": "Njoki",
    "email": "jane.njoki@leysco.co.ke",
    "role": "Senior Sales Representative",
    "permissions": ["view_sales", "create_sales"]
  }
]

2. Product Data (products.json)
[
  {
    "id": "PRD-001",
    "name": "SuperFuel Max 20W-50",
    "price": 4500.00,
    "category": "Engine Oils",
    "stock": [{"warehouse_id": "WH-001", "quantity": 150}]
  },
  {
    "id": "PRD-002",
    "name": "EcoDrive Synthetic 5W-30",
    "price": 7200.00,
    "category": "Engine Oils",
    "stock": [{"warehouse_id": "WH-001", "quantity": 200}]
  }
]

3. Customer Data (customers.json)
[
  {
    "id": "CUS-001",
    "name": "Quick Auto Services Ltd",
    "contact_person": "John Mwangi",
    "phone": "+254-712-345678",
    "email": "info@quickautoservices.co.ke"
  },
  {
    "id": "CUS-002",
    "name": "Premium Motors Kenya",
    "contact_person": "Sarah Wanjiku",
    "phone": "+254-722-678901",
    "email": "sarah.w@premiummotors.co.ke"
  }
]

4. Warehouse Data (warehouses.json)
[
  {
    "id": "WH-001",
    "name": "Nairobi Central Warehouse",
    "address": {
      "street": "Enterprise Road",
      "city": "Nairobi"
    }
  },
  {
    "id": "WH-002",
    "name": "Mombasa Regional Warehouse",
    "address": {
      "street": "Port Reitz Road",
      "city": "Mombasa"
    }
  }
]

5. Sales Data (sales_data.json)
[
  {
    "order_id": "ORD-2025-04-001",
    "customer_id": "CUS-001",
    "total_amount": 45650.00,
    "status": "Delivered"
  },
  {
    "order_id": "ORD-2025-04-002",
    "customer_id": "CUS-002",
    "total_amount": 154900.00,
    "status": "Processing"
  }
]

