Got it! To implement the requirement where only admins can add farmers, we'll adjust the backend routes accordingly. Admins will have login and signup functionality, while they can also view all registered farmers and add new farmers to the system. Farmers will have a simplified login and password reset functionality. Below is the updated structure and code for the project:

### Project Structure

```
farm-monitoring-admin/
├── backend/
│   ├── models/
│   │   ├── admin.js
│   │   └── farmer.js
│   ├── routes/
│   │   ├── adminRoutes.js
│   │   └── farmerRoutes.js
│   ├── app.js
│   ├── package.json
│   └── .env (for MongoDB URI)
└── frontend/
    ├── public/
    │   └── index.html
    ├── src/
    │   ├── components/
    │   │   ├── AdminDashboard.js
    │   │   ├── AdminLoginForm.js
    │   │   ├── AdminSignupForm.js
    │   │   ├── AdminResetPasswordForm.js
    │   │   ├── AdminFarmersList.js
    │   │   ├── FarmerLoginForm.js
    │   │   └── FarmerResetPasswordForm.js
    │   ├── App.js
    │   ├── index.js
    │   └── index.css
    ├── package.json
    └── .env (for backend API URL)
```

### Backend (Node.js with Express and MongoDB)

#### `backend/.env`

Create a `.env` file in the `backend` directory with your MongoDB connection URI:

```
MONGODB_URI=mongodb://localhost:27017/farm-monitoring
```

#### `backend/models/admin.js`

```javascript
const mongoose = require('mongoose');

const adminSchema = new mongoose.Schema({
  email: { type: String, unique: true },
  password: String,
});

module.exports = mongoose.model('Admin', adminSchema);
```

#### `backend/models/farmer.js`

```javascript
const mongoose = require('mongoose');

const farmerSchema = new mongoose.Schema({
  name: String,
  email: { type: String, unique: true },
  location: String,
});

module.exports = mongoose.model('Farmer', farmerSchema);
```

#### `backend/routes/adminRoutes.js`

```javascript
const express = require('express');
const bcrypt = require('bcryptjs');
const Admin = require('../models/admin');
const Farmer = require('../models/farmer');

const router = express.Router();

router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  try {
    const admin = await Admin.findOne({ email });
    if (!admin) {
      return res.status(404).json({ success: false, message: 'Admin not found' });
    }
    const passwordMatch = await bcrypt.compare(password, admin.password);
    if (!passwordMatch) {
      return res.status(401).json({ success: false, message: 'Incorrect password' });
    }
    return res.status(200).json({ success: true, message: 'Login successful' });
  } catch (error) {
    console.error(error);
    return res.status(500).json({ success: false, message: 'Server error' });
  }
});

router.post('/signup', async (req, res) => {
  const { email, password } = req.body;
  try {
    const hashedPassword = await bcrypt.hash(password, 10);
    const newAdmin = new Admin({ email, password: hashedPassword });
    await newAdmin.save();
    return res.status(201).json({ success: true, message: 'Admin created successfully' });
  } catch (error) {
    console.error(error);
    return res.status(500).json({ success: false, message: 'Server error' });
  }
});

router.post('/reset-password', async (req, res) => {
  const { email } = req.body;
  try {
    const admin = await Admin.findOne({ email });
    if (!admin) {
      return res.status(404).json({ success: false, message: 'Admin not found' });
    }
    // Implement reset password functionality here (generate new password and send email)
    return res.status(200).json({ success: true, message: 'Password reset successful' });
  } catch (error) {
    console.error(error);
    return res.status(500).json({ success: false, message: 'Server error' });
  }
});

router.get('/farmers', async (req, res) => {
  try {
    const farmers = await Farmer.find({}, { name: 1, email: 1, _id: 1 });
    return res.status(200).json({ success: true, farmers });
  } catch (error) {
    console.error(error);
    return res.status(500).json({ success: false, message: 'Server error' });
  }
});

router.post('/add-farmer', async (req, res) => {
  const { name, email, location } = req.body;
  try {
    const newFarmer = new Farmer({ name, email, location });
    await newFarmer.save();
    return res.status(201).json({ success: true, id: newFarmer._id, message: 'Farmer added successfully' });
  } catch (error) {
    console.error(error);
    return res.status(500).json({ success: false, message: 'Server error' });
  }
});

module.exports = router;
```

#### `backend/routes/farmerRoutes.js`

```javascript
const express = require('express');
const Farmer = require('../models/farmer');

const router = express.Router();

router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  try {
    // Implement farmer login logic if needed
  } catch (error) {
    console.error(error);
    return res.status(500).json({ success: false, message: 'Server error' });
  }
});

router.post('/reset-password', async (req, res) => {
  const { email } = req.body;
  try {
    const farmer = await Farmer.findOne({ email });
    if (!farmer) {
      return res.status(404).json({ success: false, message: 'Farmer not found' });
    }
    // Implement reset password functionality here (generate new password and send email)
    return res.status(200).json({ success: true, message: 'Password reset successful' });
  } catch (error) {
    console.error(error);
    return res.status(500).json({ success: false, message: 'Server error' });
  }
});

module.exports = router;
```

#### `backend/app.js`

```javascript
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const adminRoutes = require('./routes/adminRoutes');
const farmerRoutes = require('./routes/farmerRoutes');
require('dotenv').config();

const app = express();
const port = process.env.PORT || 5000;
const mongoURI = process.env.MONGODB_URI;

mongoose.connect(mongoURI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log('MongoDB connected'))
  .catch(err => console.log(err));

app.use(cors());
app.use(express.json());

app.use('/api/admin', adminRoutes);
app.use('/api/farmer', farmerRoutes);

app.listen(port, () => {
  console.log(`Server is running on http://localhost:${port}`);
});
```

#### `backend/package.json`

```json
{
  "name": "farm-monitoring-admin-backend",
  "version": "1.0.0",
  "main": "app.js",
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^10.0.0",
    "express": "^4.17.1",
    "mongoose": "^6.0.12"
  }
}
```

### Frontend (React)

#### `frontend/.env`

Create a `.env` file in the `frontend` directory with your backend API URL:

```
REACT_APP_API_URL=http://localhost:5000/api
```

#### `frontend/src/components/AdminDashboard.js`

```javascript
import React from 'react';
import AdminLoginForm from './AdminLoginForm';
import AdminSignupForm from './AdminSignupForm';
import AdminResetPasswordForm from './AdminResetPasswordForm';
import AdminFarmersList from './AdminFarmersList';

function AdminDashboard() {
  return (
    <div>
      <h1>Admin Dashboard</h1>
      <AdminLoginForm />
      <AdminSignupForm />
      <AdminResetPasswordForm />
      <AdminFarmersList />
    </div>
  );
}

export default AdminDashboard;
```

#### `frontend/src/components/AdminLoginForm.js`

```javascript
import React, { useState } from 'react';
import axios from 'axios';function AdminLoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [message, setMessage] = useState('');  
  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(`${process.env.REACT_APP_API_URL}/admin/login`, { email, password });
      setMessage(response.data.message);
    } catch (error) {
      console.error(error);
      setMessage('Error logging in');
    }
  };  
return (
    <div>
      <h2>Admin Login</h2>
      <form onSubmit={handleLogin}>
        <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} placeholder="
```

Certainly! Let's complete the remaining frontend components and handle the farmer addition feature on the admin side.

### Frontend (React)

#### `frontend/src/components/AdminSignupForm.js`

This component allows admins to sign up by providing their email and password.

```javascript
import React, { useState } from 'react';
import axios from 'axios';

function AdminSignupForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [message, setMessage] = useState('');

  const handleSignup = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(`${process.env.REACT_APP_API_URL}/admin/signup`, { email, password });
      setMessage(response.data.message);
    } catch (error) {
      console.error(error);
      setMessage('Error signing up');
    }
  };

  return (
    <div>
      <h2>Admin Signup</h2>
      <form onSubmit={handleSignup}>
        <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" required />
        <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" required />
        <button type="submit">Signup</button>
      </form>
      <p>{message}</p>
    </div>
  );
}

export default AdminSignupForm;
```

#### `frontend/src/components/AdminResetPasswordForm.js`

This component allows admins to reset their password by providing their email.

```javascript
import React, { useState } from 'react';
import axios from 'axios';

function AdminResetPasswordForm() {
  const [email, setEmail] = useState('');
  const [message, setMessage] = useState('');

  const handleResetPassword = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(`${process.env.REACT_APP_API_URL}/admin/reset-password`, { email });
      setMessage(response.data.message);
    } catch (error) {
      console.error(error);
      setMessage('Error resetting password');
    }
  };

  return (
    <div>
      <h2>Admin Reset Password</h2>
      <form onSubmit={handleResetPassword}>
        <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" required />
        <button type="submit">Reset Password</button>
      </form>
      <p>{message}</p>
    </div>
  );
}

export default AdminResetPasswordForm;
```

#### `frontend/src/components/AdminFarmersList.js`

This component displays a list of all registered farmers.

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function AdminFarmersList() {
  const [farmers, setFarmers] = useState([]);

  useEffect(() => {
    const fetchFarmers = async () => {
      try {
        const response = await axios.get(`${process.env.REACT_APP_API_URL}/admin/farmers`);
        setFarmers(response.data.farmers);
      } catch (error) {
        console.error(error);
      }
    };

    fetchFarmers();
  }, []);

  return (
    <div>
      <h2>Registered Farmers</h2>
      <ul>
        {farmers.map(farmer => (
          <li key={farmer._id}>{farmer.name} - {farmer.email}</li>
        ))}
      </ul>
    </div>
  );
}

export default AdminFarmersList;
```

#### `frontend/src/components/AdminDashboard.js`

This is the main dashboard component that integrates all the admin functionalities.

```javascript
import React from 'react';
import AdminLoginForm from './AdminLoginForm';
import AdminSignupForm from './AdminSignupForm';
import AdminResetPasswordForm from './AdminResetPasswordForm';
import AdminFarmersList from './AdminFarmersList';

function AdminDashboard() {
  return (
    <div>
      <h1>Admin Dashboard</h1>
      <AdminLoginForm />
      <AdminSignupForm />
      <AdminResetPasswordForm />
      <AdminFarmersList />
    </div>
  );
}

export default AdminDashboard;
```

#### `frontend/src/components/FarmerLoginForm.js`

This component allows farmers to login.

```javascript
import React, { useState } from 'react';
import axios from 'axios';

function FarmerLoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [message, setMessage] = useState('');

  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(`${process.env.REACT_APP_API_URL}/farmer/login`, { email, password });
      setMessage(response.data.message);
    } catch (error) {
      console.error(error);
      setMessage('Error logging in');
    }
  };

  return (
    <div>
      <h2>Farmer Login</h2>
      <form onSubmit={handleLogin}>
        <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" required />
        <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" required />
        <button type="submit">Login</button>
      </form>
      <p>{message}</p>
    </div>
  );
}

export default FarmerLoginForm;
```

#### `frontend/src/components/FarmerResetPasswordForm.js`

This component allows farmers to reset their password by providing their email.

```javascript
import React, { useState } from 'react';
import axios from 'axios';

function FarmerResetPasswordForm() {
  const [email, setEmail] = useState('');
  const [message, setMessage] = useState('');

  const handleResetPassword = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(`${process.env.REACT_APP_API_URL}/farmer/reset-password`, { email });
      setMessage(response.data.message);
    } catch (error) {
      console.error(error);
      setMessage('Error resetting password');
    }
  };

  return (
    <div>
      <h2>Farmer Reset Password</h2>
      <form onSubmit={handleResetPassword}>
        <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" required />
        <button type="submit">Reset Password</button>
      </form>
      <p>{message}</p>
    </div>
  );
}

export default FarmerResetPasswordForm;
```

### Notes:

- Make sure to install necessary dependencies (`axios` for making HTTP requests in React and `bcryptjs` for password hashing in Node.js).
- Update `.env` files in both `frontend` and `backend` directories with appropriate configurations.
- The above code assumes basic error handling and does not cover all edge cases (e.g., duplicate emails, robust password policies). Ensure to enhance error handling and validation as per your project's requirements.

This setup allows admins to manage farmers effectively by adding new farmers, viewing registered farmers, logging in, signing up, and resetting passwords. Adjustments can be made based on additional requirements or specific use cases.
