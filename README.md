// payment-with-promo.spec.js
describe('Payment with Promo Test', () => {
    const testUser = {
      email: 'rumi@gmail.com',
      password: 'paragontestA@'
    };
    const productName = "Collagen Drink";
    const promoCode = "QRP-TEST-123";
    const shippingAddress = {
      name: "rumi",
      phone: "628912739218",
      address: "bulan"
    };
  
    before(() => {
      // Login dan setup cart
      cy.visit('https://recruitment-staging-queenbee.paradev.io/login');
      cy.loginUser(testUser.email, testUser.password);
      cy.addProductToCart(productName, 3); // Add 3 items to meet promo requirement
      cy.applyPromoCode(promoCode);
    });
  
    it('Successfully completes payment with promo', () => {
      // 1. Navigate to checkout
      cy.visit('/checkout');
      
      // 2. Fill shipping information
      cy.fillShippingDetails(shippingAddress);
      
      // 3. Verify promo is applied
      cy.get('[data-testid="promo-applied"]')
        .should('contain', promoCode)
        .and('contain', 'Applied');
      
      // 4. Verify discounted total
      cy.get('[data-testid="subtotal-amount"]').then(($subtotal) => {
        const subtotal = parseFloat($subtotal.text().replace(/[^\d]/g, ''));
        cy.get('[data-testid="discount-amount"]').then(($discount) => {
          const discount = parseFloat($discount.text().replace(/[^\d]/g, ''));
          cy.get('[data-testid="total-amount"]').should(($total) => {
            const total = parseFloat($total.text().replace(/[^\d]/g, ''));
            expect(total).to.equal(subtotal - discount);
          });
        });
      });
      
      // 5. Select payment method (non-real payment)
      cy.selectPaymentMethod('Bank Transfer');
      
      // 6. Complete order
      cy.get('[data-testid="place-order-btn"]').click();
      
      // 7. Verify success page
      cy.url().should('include', '/order-success');
      cy.get('[data-testid="order-success"]').should('be.visible');
      cy.get('[data-testid="promo-used"]').should('contain', promoCode);
      
      // 8. Verify order in history
      cy.visit('/order-history');
      cy.get('[data-testid="order-list"]')
        .first()
        .should('contain', 'Completed')
        .and('contain', promoCode);
    });
  
    // Negative test cases
    it('Fails payment when promo requirements not met', () => {
      // Clear cart and add insufficient items
      cy.clearCart();
      cy.addProductToCart(productName, 1); // Only 1 item (not meeting promo min)
      
      cy.visit('/checkout');
      cy.get('[data-testid="promo-code-input"]').type(promoCode);
      cy.get('[data-testid="apply-promo-btn"]').click();
      
      cy.get('[data-testid="promo-error"]')
        .should('contain', 'Minimum purchase not met');
    });
  });
  
  // commands.js (Custom Commands)
  Cypress.Commands.add('loginUser', (email, password) => {
    cy.get('[data-testid="email-input"]').type(email);
    cy.get('[data-testid="password-input"]').type(password);
    cy.get('[data-testid="login-btn"]').click();
    cy.url().should('include', '/dashboard');
  });
  
  Cypress.Commands.add('addProductToCart', (productName, quantity = 1) => {
    cy.visit('/products');
    cy.searchProduct(productName);
    for (let i = 0; i < quantity; i++) {
      cy.get(`[data-testid="product-${productName}"] .add-to-cart-btn`).click();
      cy.wait(300); // Wait for cart animation
    }
  });
  
  Cypress.Commands.add('applyPromoCode', (promoCode) => {
    cy.visit('/cart');
    cy.get('[data-testid="promo-code-input"]').type(promoCode);
    cy.get('[data-testid="apply-promo-btn"]').click();
    cy.get('[data-testid="promo-success"]').should('be.visible');
  });
  
  Cypress.Commands.add('fillShippingDetails', (address) => {
    cy.get('[data-testid="shipping-name"]').type(address.name);
    cy.get('[data-testid="shipping-phone"]').type(address.phone);
    cy.get('[data-testid="shipping-address"]').type(address.address);
  });
  
  Cypress.Commands.add('selectPaymentMethod', (method) => {
    cy.get('[data-testid="payment-methods"]').select(method);
  });
  
  Cypress.Commands.add('clearCart', () => {
    cy.visit('/cart');
    cy.get('[data-testid="clear-cart-btn"]').click();
  });
