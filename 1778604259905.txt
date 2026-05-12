const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const cors = require('cors');
const { createClient } = require('@supabase/supabase-js');

const app = express();
app.use(cors());
app.use(express.json({ limit: '50mb' }));

const supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_ANON_KEY);
const JWT_SECRET = process.env.JWT_SECRET || 'user-secret-key-123';
const ADMIN_JWT_SECRET = process.env.ADMIN_JWT_SECRET || 'admin-secret-key-456';

app.get('/health', (req, res) => {
    res.status(200).json({ status: 'ok', version: '56b7c76' });
});

app.get('/', (req, res) => {
    res.json({ message: 'PKI92 Wallet API Server', version: '56b7c76' });
});

app.post('/api/register', async (req, res) => {
    try {
        const { phone, password, referralCode } = req.body;
        if (!phone || !password) {
            return res.status(400).json({ success: false, message: 'Phone aur password dono chahiye' });
        }
        const { data: existing } = await supabase.from('users').select('id').eq('phone', phone).single();
        if (existing) {
            return res.json({ success: false, message: 'Yeh phone number pehle se registered hai' });
        }
        const hashedPassword = await bcrypt.hash(password, 10);
        const newReferralCode = 'REF' + Math.random().toString(36).substring(2, 8).toUpperCase();
        let referredBy = null;
        if (referralCode) {
            const { data: referrer } = await supabase.from('users').select('id').eq('referral_code', referralCode).single();
            if (referrer) referredBy = referrer.id;
        }
        const { data: user, error } = await supabase.from('users').insert([{
            phone, password: hashedPassword, referral_code: newReferralCode, referred_by: referredBy, balance: 0
        }]).select().single();
        if (error) throw error;
        if (referredBy) {
            await supabase.rpc('add_balance', { user_id: referredBy, amount: 50 });
            await supabase.from('transactions').insert([{
                user_id: referredBy, type: 'referral_bonus', amount: 50, description: `Referral bonus from ${phone}`
            }]);
        }
        const token = jwt.sign({ userId: user.id, phone: user.phone }, JWT_SECRET, { expiresIn: '24h' });
        res.json({ success: true, token, user: { id: user.id, phone: user.phone, balance: user.balance, referralCode: newReferralCode }});
    } catch (err) {
        console.error('Register error:', err);
        res.status(500).json({ success: false, message: 'Server error: ' + err.message });
    }
});

app.post('/api/login', async (req, res) => {
    try {
        const { phone, password } = req.body;
        if (!phone || !password) {
            return res.status(400).json({ success: false, message: 'Phone aur password dono chahiye' });
        }
        const { data: user } = await supabase.from('users').select('*').eq('phone', phone).single();
        if (!user) {
            return res.json({ success: false, message: 'User nahi mila' });
        }
        const isValid = await bcrypt.compare(password, user.password);
        if (!isValid) {
            return res.json({ success: false, message: 'Galat password' });
        }
        const token = jwt.sign({ userId: user.id, phone: user.phone }, JWT_SECRET, { expiresIn: '24h' });
        res.json({ success: true, token, user: { id: user.id, phone: user.phone, balance: user.balance, referralCode: user.referral_code }});
    } catch (err) {
        console.error('Login error:', err);
        res.status(500).json({ success: false, message: 'Server error: ' + err.message });
    }
});

app.get('/api/user/profile', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        if (!token) return res.status(401).json({ success: false, message: 'Token nahi hai' });
        const decoded = jwt.verify(token, JWT_SECRET);
        const { data: user } = await supabase.from('users').select('id, phone, balance, referral_code, created_at').eq('id', decoded.userId).single();
        res.json({ success: true, user });
    } catch (err) {
        res.status(401).json({ success: false, message: 'Invalid token' });
    }
});

app.get('/api/user/balance', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        const decoded = jwt.verify(token, JWT_SECRET);
        const { data: user } = await supabase.from('users').select('balance').eq('id', decoded.userId).single();
        res.json({ success: true, balance: user?.balance || 0 });
    } catch (err) {
        res.status(401).json({ success: false, message: 'Invalid token' });
    }
});

app.get('/api/user/transactions', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        const decoded = jwt.verify(token, JWT_SECRET);
        const { data: transactions } = await supabase.from('transactions').select('*').eq('user_id', decoded.userId).order('created_at', { ascending: false });
        res.json({ success: true, transactions });
    } catch (err) {
        res.status(401).json({ success: false, message: 'Invalid token' });
    }
});

app.post('/api/deposit', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        const decoded = jwt.verify(token, JWT_SECRET);
        const { amount, utrNumber, screenshotUrl } = req.body;
        if (!amount || amount <= 0) {
            return res.status(400).json({ success: false, message: 'Valid amount daalo' });
        }
        const { data: deposit } = await supabase.from('deposits').insert([{
            user_id: decoded.userId, amount, utr_number: utrNumber, screenshot_url: screenshotUrl, status: 'pending'
        }]).select().single();
        res.json({ success: true, message: 'Deposit request admin ke paas bhej di gayi hai', depositId: deposit.id });
    } catch (err) {
        res.status(500).json({ success: false, message: err.message });
    }
});

app.post('/api/withdraw', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        const decoded = jwt.verify(token, JWT_SECRET);
        const { amount, bankName, accountNumber, ifscCode, accountHolder } = req.body;
        if (!amount || amount <= 0) {
            return res.status(400).json({ success: false, message: 'Valid amount daalo' });
        }
        const { data: user } = await supabase.from('users').select('balance').eq('id', decoded.userId).single();
        if (user.balance < amount) {
            return res.json({ success: false, message: 'Balance kam hai' });
        }
        const { data: withdrawal } = await supabase.from('withdrawals').insert([{
            user_id: decoded.userId, amount, bank_name: bankName, account_number: accountNumber, ifsc_code: ifscCode, account_holder: accountHolder, status: 'pending'
        }]).select().single();
        await supabase.rpc('deduct_balance', { user_id: decoded.userId, amount: amount });
        res.json({ success: true, message: 'Withdrawal request admin ke paas bhej di gayi hai', withdrawalId: withdrawal.id });
    } catch (err) {
        res.status(500).json({ success: false, message: err.message });
    }
});

app.post('/api/admin/login', async (req, res) => {
    try {
        const { username, password } = req.body;
        const { data: admin } = await supabase.from('admins').select('*').eq('username', username).single();
        if (!admin) {
            return res.json({ success: false, message: 'Admin nahi mila' });
        }
        const isValid = await bcrypt.compare(password, admin.password);
        if (!isValid) {
            return res.json({ success: false, message: 'Galat password' });
        }
        const token = jwt.sign({ adminId: admin.id, role: admin.role }, ADMIN_JWT_SECRET, { expiresIn: '24h' });
        res.json({ success: true, token, admin: { username: admin.username, role: admin.role }});
    } catch (err) {
        res.status(500).json({ success: false, message: err.message });
    }
});

app.get('/api/admin/dashboard', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        jwt.verify(token, ADMIN_JWT_SECRET);
        const { data: users } = await supabase.from('users').select('id, balance');
        const { data: pendingDeposits } = await supabase.from('deposits').select('id').eq('status', 'pending');
        const { data: pendingWithdrawals } = await supabase.from('withdrawals').select('id').eq('status', 'pending');
        const totalBalance = users?.reduce((sum, u) => sum + parseFloat(u.balance), 0) || 0;
        res.json({ success: true, stats: { totalUsers: users?.length || 0, totalBalance, pendingDeposits: pendingDeposits?.length || 0, pendingWithdrawals: pendingWithdrawals?.length || 0 }});
    } catch (err) {
        res.status(401).json({ success: false, message: 'Admin access chahiye' });
    }
});

app.get('/api/admin/users', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        jwt.verify(token, ADMIN_JWT_SECRET);
        const { data: users } = await supabase.from('users').select('id, phone, balance, referral_code, status, created_at').order('created_at', { ascending: false });
        res.json({ success: true, users, total: users?.length || 0 });
    } catch (err) {
        res.status(401).json({ success: false, message: 'Admin access chahiye' });
    }
});

app.get('/api/admin/deposits', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        jwt.verify(token, ADMIN_JWT_SECRET);
        const { data: deposits } = await supabase.from('deposits').select('*, users(phone)').order('created_at', { ascending: false });
        res.json({ success: true, deposits });
    } catch (err) {
        res.status(401).json({ success: false, message: 'Admin access chahiye' });
    }
});

app.post('/api/admin/deposit/:id', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        const decoded = jwt.verify(token, ADMIN_JWT_SECRET);
        const { id } = req.params;
        const { status } = req.body;
        const { data: deposit } = await supabase.from('deposits').select('*').eq('id', id).single();
        if (status === 'approved') {
            await supabase.rpc('add_balance', { user_id: deposit.user_id, amount: deposit.amount });
            await supabase.from('transactions').insert([{ user_id: deposit.user_id, type: 'deposit', amount: deposit.amount, description: 'Deposit approved by admin' }]);
        }
        await supabase.from('deposits').update({ status, approved_by: decoded.adminId, approved_at: new Date() }).eq('id', id);
        res.json({ success: true, message: `Deposit ${status} kar diya gaya` });
    } catch (err) {
        res.status(500).json({ success: false, message: err.message });
    }
});

app.get('/api/admin/withdrawals', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        jwt.verify(token, ADMIN_JWT_SECRET);
        const { data: withdrawals } = await supabase.from('withdrawals').select('*, users(phone)').order('created_at', { ascending: false });
        res.json({ success: true, withdrawals });
    } catch (err) {
        res.status(401).json({ success: false, message: 'Admin access chahiye' });
    }
});

app.post('/api/admin/withdrawal/:id', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        const decoded = jwt.verify(token, ADMIN_JWT_SECRET);
        const { id } = req.params;
        const { status } = req.body;
        const { data: withdrawal } = await supabase.from('withdrawals').select('*').eq('id', id).single();
        if (status === 'rejected') {
            await supabase.rpc('add_balance', { user_id: withdrawal.user_id, amount: withdrawal.amount });
        }
        if (status === 'approved') {
            await supabase.from('transactions').insert([{ user_id: withdrawal.user_id, type: 'withdrawal', amount: -withdrawal.amount, description: 'Withdrawal approved by admin' }]);
        }
        await supabase.from('withdrawals').update({ status, approved_by: decoded.adminId, approved_at: new Date() }).eq('id', id);
        res.json({ success: true, message: `Withdrawal ${status} kar diya gaya` });
    } catch (err) {
        res.status(500).json({ success: false, message: err.message });
    }
});

app.get('/api/admin/transactions', async (req, res) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        jwt.verify(token, ADMIN_JWT_SECRET);
        const { data: transactions } = await supabase.from('transactions').select('*, users(phone)').order('created_at', { ascending: false });
        res.json({ success: true, transactions });
    } catch (err) {
        res.status(401).json({ success: false, message: 'Admin access chahiye' });
    }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`PKI92 Server v56b7c76 chal raha hai port ${PORT} par`);
});
{
  "name": "pki92-wallet-backend",
  "version": "56b7c76",
  "description": "PKI92 Wallet - Backend API Server",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "@supabase/supabase-js": "^2.39.0",
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.2"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
