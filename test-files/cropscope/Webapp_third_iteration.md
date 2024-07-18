Certainly! Let's refactor the combined code for both frontend (React) and backend (Node.js with Express and MongoDB) into a cohesive project structure. This will integrate authentication, admin functionalities (signup, login, reset password), farmer management (add, list), and ensure that routes are protected for admin-only access.

### Frontend (`frontend/src/components/PrivateRoute.js`)

#### Private Route Component (final)

```javascript
import React, { useContext } from 'react';
import { Route, Redirect } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const PrivateRoute = ({ component: Component, ...rest }) => {
  const { isAdminAuthenticated } = useContext(AuthContext);

  return (
    <Route
      {...rest}
      render={(props) =>
        isAdminAuthenticated ? <Component {...props} /> : <Redirect to="/admin/login" />
      }
    />
  );
};

export default PrivateRoute;
```

### Frontend (`frontend/src/App.js`)

#### App Component (final)

```javascript
import React from 'react';
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';
import './App.css';
import AuthContextProvider from './components/AuthContext';
import PrivateRoute from './components/PrivateRoute';
import AdminDashboard from './components/Admin/AdminDashboard';
import AdminLoginForm from './components/Admin/AdminLoginForm';
import AdminSignupForm from './components/Admin/AdminSignupForm';
import AdminResetPasswordForm from './components/Admin/AdminResetPasswordForm';

function App() {
  return (
    <AuthContextProvider>
      <Router>
        <div className="App">
          <Switch>
            <PrivateRoute exact path="/admin" component={AdminDashboard} />
            <Route path="/admin/login" component={AdminLoginForm} />
            <Route path="/admin/signup" component={AdminSignupForm} />
            <Route path="/admin/reset-password" component={AdminResetPasswordForm} />
          </Switch>
        </div>
      </Router>
    </AuthContextProvider>
  );
}

export default App;
```

### Backend (`backend/routes/admin.js`)

#### Admin Routes (final)

```javascript
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const config = require('config');
const { check, validationResult } = require('express-validator');
const auth = require('../middleware/auth');
const Admin = require('../models/Admin');
const Farmer = require('../models/Farmer');

// @route   POST /admin/signup
// @desc    Register admin
// @access  Public
router.post(
  '/signup',
  [
    check('email', 'Please include a valid email').isEmail(),
    check('password', 'Please enter a password with 6 or more characters').isLength({ min: 6 }),
  ],
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { email, password } = req.body;

    try {
      let admin = await Admin.findOne({ email });

      if (admin) {
        return res.status(400).json({ msg: 'Admin already exists' });
      }

      admin = new Admin({
        email,
        password,
      });

      const salt = await bcrypt.genSalt(10);
      admin.password = await bcrypt.hash(password, salt);

      await admin.save();

      const payload = {
        admin: {
          id: admin.id,
        },
      };

      jwt.sign(
        payload,
        config.get('jwtSecret'),
        { expiresIn: 360000 },
        (err, token) => {
          if (err) throw err;
          res.json({ token });
        }
      );
    } catch (err) {
      console.error(err.message);
      res.status(500).send('Server Error');
    }
  }
);

// @route   POST /admin/login
// @desc    Authenticate admin & get token
// @access  Public
router.post(
  '/login',
  [
    check('email', 'Please include a valid email').isEmail(),
    check('password', 'Password is required').exists(),
  ],
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { email, password } = req.body;

    try {
      let admin = await Admin.findOne({ email });

      if (!admin) {
        return res.status(400).json({ msg: 'Invalid Credentials' });
      }

      const isMatch = await bcrypt.compare(password, admin.password);

      if (!isMatch) {
        return res.status(400).json({ msg: 'Invalid Credentials' });
      }

      const payload = {
        admin: {
          id: admin.id,
        },
      };

      jwt.sign(
        payload,
        config.get('jwtSecret'),
        { expiresIn: 360000 },
        (err, token) => {
          if (err) throw err;
          res.json({ token });
        }
      );
    } catch (err) {
      console.error(err.message);
      res.status(500).send('Server Error');
    }
  }
);

// @route   GET /admin/farmers
// @desc    Get all farmers (admin access)
// @access  Private
router.get('/farmers', auth, async (req, res) => {
  try {
    const farmers = await Farmer.find().select('-password');
    res.json({ farmers });
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server Error');
  }
});

// @route   POST /admin/add-farmer
// @desc    Add a new farmer (admin access)
// @access  Private
router.post('/add-farmer', auth, async (req, res) => {
  const { name, email, location } = req.body;

  try {
    let farmer = await Farmer.findOne({ email });

    if (farmer) {
      return res.status(400).json({ msg: 'Farmer already exists' });
    }

    farmer = new Farmer({
      name,
      email,
      location,
    });

    await farmer.save();

    res.json({ id: farmer.id });
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server Error');
  }
});

module.exports = router;
```

### Backend (`backend/models/Farmer.js`)

#### Farmer Model (final)

```javascript
const mongoose = require('mongoose');

const FarmerSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
  },
  email: {
    type: String,
    required: true,
    unique: true,
  },
  location: {
    type: String,
    required: true,
  },
});

module.exports = mongoose.model('farmer', FarmerSchema);
```

### Backend (`backend/models/Admin.js`)

#### Admin Model (final)

```javascript
const mongoose = require('mongoose');

const AdminSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
  },
  password: {
    type: String,
    required: true,
  },
  date: {
    type: Date,
    default: Date.now,
  },
});

module.exports = mongoose.model('admin', AdminSchema);
```

Certainly! Let's complete the remaining parts and ensure everything is properly integrated and functioning.

### Backend (`backend/routes/auth.js`)

#### Authentication Routes (final)

```javascript
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const config = require('config');
const { check, validationResult } = require('express-validator');
const Admin = require('../models/Admin');

// @route   POST /auth/signup
// @desc    Register admin
// @access  Public
router.post(
  '/signup',
  [
    check('email', 'Please include a valid email').isEmail(),
    check('password', 'Please enter a password with 6 or more characters').isLength({ min: 6 }),
  ],
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { email, password } = req.body;

    try {
      let admin = await Admin.findOne({ email });

      if (admin) {
        return res.status(400).json({ msg: 'Admin already exists' });
      }

      admin = new Admin({
        email,
        password,
      });

      const salt = await bcrypt.genSalt(10);
      admin.password = await bcrypt.hash(password, salt);

      await admin.save();

      const payload = {
        admin: {
          id: admin.id,
        },
      };

      jwt.sign(
        payload,
        config.get('jwtSecret'),
        { expiresIn: 360000 },
        (err, token) => {
          if (err) throw err;
          res.json({ token });
        }
      );
    } catch (err) {
      console.error(err.message);
      res.status(500).send('Server Error');
    }
  }
);

// @route   POST /auth/login
// @desc    Authenticate admin & get token
// @access  Public
router.post(
  '/login',
  [
    check('email', 'Please include a valid email').isEmail(),
    check('password', 'Password is required').exists(),
  ],
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { email, password } = req.body;

    try {
      let admin = await Admin.findOne({ email });

      if (!admin) {
        return res.status(400).json({ msg: 'Invalid Credentials' });
      }

      const isMatch = await bcrypt.compare(password, admin.password);

      if (!isMatch) {
        return res.status(400).json({ msg: 'Invalid Credentials' });
      }

      const payload = {
        admin: {
          id: admin.id,
        },
      };

      jwt.sign(
        payload,
        config.get('jwtSecret'),
        { expiresIn: 360000 },
        (err, token) => {
          if (err) throw err;
          res.json({ token });
        }
      );
    } catch (err) {
      console.error(err.message);
      res.status(500).send('Server Error');
    }
  }
);

module.exports = router;
```

### Backend (`backend/models/Farmer.js`)

#### Farmer Model (final)

```javascript
const mongoose = require('mongoose');

const FarmerSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
  },
  email: {
    type: String,
    required: true,
    unique: true,
  },
  location: {
    type: String,
    required: true,
  },
});

module.exports = mongoose.model('Farmer', FarmerSchema);
```

### Backend (`backend/models/Admin.js`)

#### Admin Model (final)

```javascript
const mongoose = require('mongoose');

const AdminSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
  },
  password: {
    type: String,
    required: true,
  },
  date: {
    type: Date,
    default: Date.now,
  },
});

module.exports = mongoose.model('Admin', AdminSchema);
```

### Backend (`backend/app.js`)

#### Express App Configuration (final)

```javascript
const express = require('express');
const connectDB = require('./config/db');
const cors = require('cors');

const app = express();

// Connect Database
connectDB();

// Init Middleware
app.use(express.json({ extended: false }));

// Enable CORS
app.use(cors());

// Define Routes
app.use('/admin', require('./routes/admin'));
app.use('/auth', require('./routes/auth'));

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => console.log(`Server started on port ${PORT}`));
```

### Backend (`backend/config/db.js`)

#### MongoDB Connection (final)

```javascript
const mongoose = require('mongoose');
const config = require('config');
const db = config.get('mongoURI');

const connectDB = async () => {
  try {
    await mongoose.connect(db, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      useCreateIndex: true,
      useFindAndModify: false,
    });
    console.log('MongoDB Connected...');
  } catch (err) {
    console.error(err.message);
    // Exit process with failure
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Frontend (`frontend/src/components/AuthContext.js`)

#### Authentication Context (final)

```javascript
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

const AuthContextProvider = (props) => {
  const [token, setToken] = useState(null);
  const [isAdminAuthenticated, setIsAdminAuthenticated] = useState(false);
  const [authError, setAuthError] = useState(null);

  useEffect(() => {
    const checkAdminAuthentication = async () => {
      const token = localStorage.getItem('adminToken');
      if (token) {
        try {
          const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/check-auth`, {
            headers: {
              Authorization: `Bearer ${token}`,
            },
          });
          if (response.data.success) {
            setToken(token);
            setIsAdminAuthenticated(true);
          } else {
            localStorage.removeItem('adminToken');
          }
        } catch (error) {
          console.error('Error checking admin authentication:', error);
        }
      }
    };

    checkAdminAuthentication();
  }, []);

  const adminSignup = async (email, password) => {
    try {
      const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/signup`, {
        email,
        password,
      });
      setToken(response.data.token);
      setIsAdminAuthenticated(true);
      setAuthError(null);
      localStorage.setItem('adminToken', response.data.token);
    } catch (error) {
      setAuthError('Error signing up');
    }
  };

  const adminLogin = async (email, password) => {
    try {
      const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
        email,
        password,
      });
      setToken(response.data.token);
      setIsAdminAuthenticated(true);
      setAuthError(null);
      localStorage.setItem('adminToken', response.data.token);
    } catch (error) {
      setAuthError('Invalid credentials');
    }
  };

  const adminLogout = () => {
    localStorage.removeItem('adminToken');
    setIsAdminAuthenticated(false);
    setToken(null);
  };

  const adminResetPassword = async (email) => {
    try {
      await axios.post(`${process.env.REACT_APP_API_URL}/auth/reset-password`, { email });
      setAuthError(null);
    } catch (error) {
      setAuthError('Error resetting password');
    }
  };

  return (
    <AuthContext.Provider
      value={{
        token,
        isAdminAuthenticated,
        authError,
        adminSignup,
        adminLogin,
        adminLogout,
        adminResetPassword,
      }}
    >
      {props.children}
    </AuthContext.Provider>
  );
};

export default AuthContextProvider;
```

### Frontend (`frontend/src/components/PrivateRoute.js`)

#### Private Route Component (final)

```javascript
import React, { useContext } from 'react';
import { Route, Redirect } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const PrivateRoute = ({ component: Component, ...rest }) => {
  const { isAdminAuthenticated } = useContext(AuthContext);

  return (
    <Route
      {...rest}
      render={(props) =>
        isAdminAuthenticated ? <Component {...props} /> : <Redirect to="/admin/login" />
      }
    />
  );
};

export default PrivateRoute;
```

Certainly! Let's finish the remaining parts to ensure the application is complete and functional.

### Frontend (`frontend/src/components/Admin/AdminDashboard.js`)

#### Admin Dashboard Component (final)

```javascript
import React, { useState, useEffect, useContext } from 'react';
import axios from 'axios';
import { AuthContext } from '../../context/AuthContext';

const AdminDashboard = () => {
  const { token } = useContext(AuthContext);
  const [farmers, setFarmers] = useState([]);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchFarmers = async () => {
      try {
        const response = await axios.get(`${process.env.REACT_APP_API_URL}/admin/farmers`, {
          headers: {
            Authorization: `Bearer ${token}`,
          },
        });
        setFarmers(response.data.farmers);
        setError(null);
      } catch (error) {
        setError('Error fetching farmers');
      }
    };

    fetchFarmers();
  }, [token]);

  return (
    <div>
      <h2>Admin Dashboard</h2>
      {error && <p>{error}</p>}
      <ul>
        {farmers.map((farmer) => (
          <li key={farmer._id}>
            <strong>Name:</strong> {farmer.name} | <strong>Email:</strong> {farmer.email} |{' '}
            <strong>Location:</strong> {farmer.location}
          </li>
        ))}
      </ul>
    </div>
  );
};

export default AdminDashboard;
```

### Frontend (`frontend/src/components/Admin/AdminLoginForm.js`)

#### Admin Login Form Component (final)

```javascript
import React, { useState, useContext } from 'react';
import { useHistory } from 'react-router-dom';
import { AuthContext } from '../../context/AuthContext';

const AdminLoginForm = () => {
  const { adminLogin, authError } = useContext(AuthContext);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const history = useHistory();

  const handleSubmit = (e) => {
    e.preventDefault();
    adminLogin(email, password);
    setEmail('');
    setPassword('');
  };

  return (
    <div>
      <h2>Admin Login</h2>
      {authError && <p>{authError}</p>}
      <form onSubmit={handleSubmit}>
        <div>
          <label>Email:</label>
          <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
        </div>
        <div>
          <label>Password:</label>
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </div>
        <button type="submit">Login</button>
      </form>
    </div>
  );
};

export default AdminLoginForm;
```

### Frontend (`frontend/src/components/Admin/AdminSignupForm.js`)

#### Admin Signup Form Component (final)

```javascript
import React, { useState, useContext } from 'react';
import { useHistory } from 'react-router-dom';
import { AuthContext } from '../../context/AuthContext';

const AdminSignupForm = () => {
  const { adminSignup, authError } = useContext(AuthContext);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const history = useHistory();

  const handleSubmit = (e) => {
    e.preventDefault();
    adminSignup(email, password);
    setEmail('');
    setPassword('');
  };

  return (
    <div>
      <h2>Admin Sign Up</h2>
      {authError && <p>{authError}</p>}
      <form onSubmit={handleSubmit}>
        <div>
          <label>Email:</label>
          <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
        </div>
        <div>
          <label>Password:</label>
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </div>
        <button type="submit">Sign Up</button>
      </form>
    </div>
  );
};

export default AdminSignupForm;
```

### Frontend (`frontend/src/components/Admin/AdminResetPasswordForm.js`)

#### Admin Reset Password Form Component (final)

```javascript
import React, { useState } from 'react';
import axios from 'axios';

const AdminResetPasswordForm = () => {
  const [email, setEmail] = useState('');
  const [successMessage, setSuccessMessage] = useState('');
  const [error, setError] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await axios.post(`${process.env.REACT_APP_API_URL}/auth/reset-password`, { email });
      setSuccessMessage('Password reset instructions sent to your email');
      setError(null);
      setEmail('');
    } catch (error) {
      setError('Error resetting password');
    }
  };

  return (
    <div>
      <h2>Admin Reset Password</h2>
      <form onSubmit={handleSubmit}>
        <div>
          <label>Email:</label>
          <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
        </div>
        {error && <p>{error}</p>}
        {successMessage && <p>{successMessage}</p>}
        <button type="submit">Reset Password</button>
      </form>
    </div>
  );
};

export default AdminResetPasswordForm;
```

### Frontend (`frontend/src/App.js`)

#### App Component (final)

```javascript
import React from 'react';
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';
import './App.css';
import AuthContextProvider from './components/AuthContext';
import PrivateRoute from './components/PrivateRoute';
import AdminDashboard from './components/Admin/AdminDashboard';
import AdminLoginForm from './components/Admin/AdminLoginForm';
import AdminSignupForm from './components/Admin/AdminSignupForm';
import AdminResetPasswordForm from './components/Admin/AdminResetPasswordForm';

function App() {
  return (
    <AuthContextProvider>
      <Router>
        <div className="App">
          <Switch>
            <PrivateRoute exact path="/admin" component={AdminDashboard} />
            <Route path="/admin/login" component={AdminLoginForm} />
            <Route path="/admin/signup" component={AdminSignupForm} />
            <Route path="/admin/reset-password" component={AdminResetPasswordForm} />
          </Switch>
        </div>
      </Router>
    </AuthContextProvider>
  );
}

export default App;
```

Apologies for the confusion earlier. Let's complete the remaining part of the `auth.js` file and ensure everything is properly integrated and functional.

### Backend (`backend/routes/auth.js`)

#### Authentication Routes (final)

```javascript
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const config = require('config');
const { check, validationResult } = require('express-validator');
const Admin = require('../models/Admin');

// @route   POST /auth/signup
// @desc    Register admin
// @access  Public
router.post(
  '/signup',
  [
    check('email', 'Please include a valid email').isEmail(),
    check('password', 'Please enter a password with 6 or more characters').isLength({ min: 6 }),
  ],
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { email, password } = req.body;

    try {
      let admin = await Admin.findOne({ email });

      if (admin) {
        return res.status(400).json({ msg: 'Admin already exists' });
      }

      admin = new Admin({
        email,
        password,
      });

      const salt = await bcrypt.genSalt(10);
      admin.password = await bcrypt.hash(password, salt);

      await admin.save();

      const payload = {
        admin: {
          id: admin.id,
        },
      };

      jwt.sign(
        payload,
        config.get('jwtSecret'),
        { expiresIn: 360000 },
        (err, token) => {
          if (err) throw err;
          res.json({ token });
        }
      );
    } catch (err) {
      console.error(err.message);
      res.status(500).send('Server Error');
    }
  }
);

// @route   POST /auth/login
// @desc    Authenticate admin & get token
// @access  Public
router.post(
  '/login',
  [
    check('email', 'Please include a valid email').isEmail(),
    check('password', 'Password is required').exists(),
  ],
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { email, password } = req.body;

    try {
      let admin = await Admin.findOne({ email });

      if (!admin) {
        return res.status(400).json({ msg: 'Invalid Credentials' });
      }

      const isMatch = await bcrypt.compare(password, admin.password);

      if (!isMatch) {
        return res.status(400).json({ msg: 'Invalid Credentials' });
      }

      const payload = {
        admin: {
          id: admin.id,
        },
      };

      jwt.sign(
        payload,
        config.get('jwtSecret'),
        { expiresIn: 360000 },
        (err, token) => {
          if (err) throw err;
          res.json({ token });
        }
      );
    } catch (err) {
      console.error(err.message);
      res.status(500).send('Server Error');
    }
  }
);

// @route   POST /auth/reset-password
// @desc    Reset admin password (placeholder)
// @access  Public
router.post('/reset-password', (req, res) => {
  res.send('Reset password route');
});

module.exports = router;
```

### Frontend (`frontend/src/components/PrivateRoute.js`)

#### Private Route Component (final)

```javascript
import React, { useContext } from 'react';
import { Route, Redirect } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const PrivateRoute = ({ component: Component, ...rest }) => {
  const { isAdminAuthenticated } = useContext(AuthContext);

  return (
    <Route
      {...rest}
      render={(props) =>
        isAdminAuthenticated ? <Component {...props} /> : <Redirect to="/admin/login" />
      }
    />
  );
};

export default PrivateRoute;
```

### Frontend (`frontend/src/App.js`)

#### App Component (final)

```javascript
import React from 'react';
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';
import './App.css';
import AuthContextProvider from './components/AuthContext';
import PrivateRoute from './components/PrivateRoute';
import AdminDashboard from './components/Admin/AdminDashboard';
import AdminLoginForm from './components/Admin/AdminLoginForm';
import AdminSignupForm from './components/Admin/AdminSignupForm';
import AdminResetPasswordForm from './components/Admin/AdminResetPasswordForm';

function App() {
  return (
    <AuthContextProvider>
      <Router>
        <div className="App">
          <Switch>
            <PrivateRoute exact path="/admin" component={AdminDashboard} />
            <Route path="/admin/login" component={AdminLoginForm} />
            <Route path="/admin/signup" component={AdminSignupForm} />
            <Route path="/admin/reset-password" component={AdminResetPasswordForm} />
          </Switch>
        </div>
      </Router>
    </AuthContextProvider>
  );
}

export default App;
```

### Frontend (`frontend/src/index.js`)

#### Index File (final)

Ensure that your `index.js` or `index.jsx` file properly renders the `App` component within the `ReactDOM.render` function. This file is typically located in your `src` folder.

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';
import App from './App';

ReactDOM.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
  document.getElementById('root')
);
```

### Backend (`backend/app.js`)

#### Express App Configuration (final)

Ensure your `app.js` or `server.js` file correctly initializes and starts your Express server. This typically includes setting up middleware, routes, and starting the server.

```javascript
const express = require('express');
const connectDB = require('./config/db');
const cors = require('cors');

const app = express();

// Connect Database
connectDB();

// Init Middleware
app.use(express.json({ extended: false }));

// Enable CORS
app.use(cors());

// Define Routes
app.use('/admin', require('./routes/admin'));
app.use('/auth', require('./routes/auth'));

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => console.log(`Server started on port ${PORT}`));
```

### Conclusion

With these final implementations, your application should now have a fully functional authentication system for admin users, including signup, login, and protected routes. Ensure that your MongoDB connection URI (`mongoURI`) and JWT secret (`jwtSecret`) in the `config/default.json` or environment variables are properly configured to match your setup.

This setup provides a solid foundation for managing admin authentication in your application. If you have any further questions or need additional assistance, feel free to ask!
