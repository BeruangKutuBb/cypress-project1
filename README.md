// registration.spec.js
describe('Account Registration Test', () => {
    beforeEach(() => {
      cy.visit('https://recruitment-staging-queenbee.paradev.io/register') 
    })
  
    it('Successfully registers a new account with valid WhatsApp number', () => {
      // Test data
      const testPhoneNumber = '1234567890' 
      const verificationCode = '123456' 
  
      // 1. Isi form pendaftaran
      cy.get('#phone-input').type(testPhoneNumber) 
      cy.get('#register-button').click()
  
      // 2. Verifikasi masuk ke halaman OTP
      cy.url().should('include', '/verify')
      cy.get('.verification-title').should('contain', 'Verification Code')
  
      // 3. Masukkan kode verifikasi
      cy.get('#verification-code-input').type(verificationCode)
      cy.get('#verify-button').click()
  
      // 4. Verifikasi registrasi berhasil
      cy.url().should('include', '/dashboard') /
      cy.get('.welcome-message').should('contain', 'Welcome')
      
      // 5. Verifikasi data user tersimpan
      cy.window().then((win) => {
        expect(win.localStorage.getItem('userToken')).to.exist
      })
    })
  
  })


  // add-to-cart.spec.js
describe('Product Addition Test', () => {
  const productName = "Collagen Drink" // Produk yang diperlukan untuk promo
  const promoCode = "QRP-TEST-123" // Kode promo dari requirement

  beforeEach(() => {
    // Login terlebih dahulu (asumsi sudah ada command login)
    cy.visit('https://recruitment-staging-queenbee.paradev.io/login')
    cy.loginUser('testuser@example.com', 'password123') // Ganti dengan account login yang sesuai
    cy.visit('/products') // diarahkan ke halaman produk
  })

  it('Successfully adds product to cart', () => {
    // 1. Cari dan tambahkan produk
    cy.searchProduct(productName)
    cy.get([data-testid="product-${productName}"]) 
      .find('.add-to-cart-btn')
      .click()

    // 2. Verifikasi produk ditambahkan
    cy.get('.cart-notification')
      .should('contain', ${productName} added to cart)
    
    cy.get('[data-testid="cart-count"]')
      .should('contain', '1')

    // 3. Verifikasi di halaman cart
    cy.visit('/cart')
    cy.get('.cart-item')
      .should('contain', productName)
      .and('contain', '1') // Qty = 1
  })

  it('Adds product and applies promo successfully', () => {
    const quantityToAdd = 3 // Asumsi perlu 3 item untuk memenuhi min purchase
    
    for (let i = 0; i < quantityToAdd; i++) {
      cy.get([data-testid="product-${productName}"])
        .find('.add-to-cart-btn')
        .click()
      cy.wait(500) // Tunggu animasi cart
    }

    // 2. Apply promo code
    cy.visit('/cart')
    cy.get('#promo-code-input').type(promoCode)
    cy.get('#apply-promo-btn').click()

    // 3. Verifikasi promo applied
    cy.get('.promo-success-message')
      .should('contain', 'Promo applied successfully')
    cy.get('.cart-total')
      .should('contain', '1.250.000') // Verifikasi discount diterapkan
  })
})

// commands.js (Custom Commands)
Cypress.Commands.add('loginUser', (email, password) => {
  cy.get('#email').type(email)
  cy.get('#password').type(password)
  cy.get('#login-btn').click()
  cy.url().should('include', '/dashboard')
})

Cypress.Commands.add('searchProduct', (productName) => {
  cy.get('#search-input').type(productName)
  cy.get('#search-btn').click()
  cy.get('.search-results').should('contain', productName)
})


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
